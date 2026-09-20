# Module: Registration

## Shared principle
Phone is the login credential, NOT a unique identity key. One phone may link multiple role-profiles (Farmer, Consumer, etc.) under the same or different Accounts. On login, if a phone has multiple linked profiles, present a role-switch screen before entering the app.

## Entity: FPO_PROFILE
| Field | Type | Notes |
|---|---|---|
| id | UUID PK | |
| account_id | UUID FK -> ACCOUNT | |
| name | string | |
| reg_number | string | format-checked only (CIN pattern or coop-society ID pattern); real MCA cross-check deferred to production |
| address | string | |
| member_count | int | |
| bank_upi_id | string | |
| crops_handled | string[] | |
| jurisdiction | string | state/district/mandi |
| status | enum [pending, active, rejected] | |
| trust_profile_id | UUID FK -> TrustProfile | |

### Rules
```
ON fpo.register:
  IF reg_number fails format check: REJECT (400, "invalid_reg_number")
  IF reg_number already exists on another FPO_PROFILE: REJECT (409, "duplicate_registration")
  fpo.status = "pending"
  create_trust_profile(fpo, base_score=50)
  enqueue_admin_review(fpo)

ON admin_approve(fpo): fpo.status = "active"
ON admin_reject(fpo, reason): fpo.status = "rejected"; notify(fpo, reason)

-- access control while pending:
IF fpo.status == "pending": ALLOW read-only browse (listings, forecasts). DENY create_listing, accept_order.
```

## Entity: FARMER_PROFILE
| Field | Type | Notes |
|---|---|---|
| id | UUID PK | |
| account_id | UUID FK -> ACCOUNT | |
| fpo_id | UUID, nullable FK -> FPO_PROFILE | |
| name | string | |
| aadhaar_hash | string | SHA-256 or equivalent hash only; raw Aadhaar never persisted, per DPDP Act 2023 |
| gps_location | geo | |
| bank_upi_id | string | |
| status | enum [pending, active, rejected] | |
| trust_profile_id | UUID FK -> TrustProfile | |

### Rules
```
ON farmer.register:
  IF aadhaar fails Verhoeff checksum: REJECT (400, "invalid_aadhaar_format")
  -- NOTE: this is format validation only, NOT real UIDAI e-KYC (requires licensed AUA/KUA access,
  -- out of scope for MVP -- document as production integration point)
  aadhaar_hash = hash(aadhaar_number)  -- raw number discarded immediately after hashing
  IF aadhaar_hash already linked to a DIFFERENT account_id:
    flag_for_manual_review(farmer, "aadhaar_phone_mismatch")  -- do not auto-accept or auto-reject
  ELSE:
    farmer.status = "pending"
  create_trust_profile(farmer, base_score=50)
  enqueue_admin_review(farmer)

-- no quantity cap by farmer-vs-FPO identity type -- see 03-listing.md; risk is managed via
-- trust_profile.offer_fulfillment_rate / completed_order_count, not identity restriction
```

## Entity: CONSUMER_PROFILE
| Field | Type | Notes |
|---|---|---|
| id | UUID PK | |
| account_id | UUID FK -> ACCOUNT | |
| name | string | |
| delivery_addresses | Address[] | one-to-many |
| payment_method_token | string | tokenized via payment gateway; raw card/UPI never stored |
| status | enum [active] | no pending state |

### Rules
```
ON consumer.register:
  verify_otp(phone)
  consumer.status = "active"  -- immediate, no admin review
  create_trust_profile(consumer, base_score=50)  -- tracks order_no_show_rate only
```

## Entity: BULKBUYER_PROFILE
| Field | Type | Notes |
|---|---|---|
| id | UUID PK | |
| account_id | UUID FK -> ACCOUNT | |
| business_name | string | |
| gstin | string | 15-char format + checksum validated |
| business_address | string | |
| payment_terms | enum [upfront_only] | credit terms deferred to future scope |
| status | enum [pending, active, rejected] | |
| trust_profile_id | UUID FK -> TrustProfile | |

### Rules
```
ON bulkbuyer.register:
  IF gstin fails format+checksum: REJECT (400, "invalid_gstin")
  bulkbuyer.status = "pending"   -- gated, unlike Consumer, due to bulk transaction value risk
  create_trust_profile(bulkbuyer, base_score=50)
  enqueue_admin_review(bulkbuyer)

-- access control while pending: read-only browse only, cannot post Requirement or place bulk Order
```

## Entity: LOGISTICS_PROFILE
| Field | Type | Notes |
|---|---|---|
| id | UUID PK | |
| account_id | UUID FK -> ACCOUNT | |
| type | enum [individual_driver, transport_company] | |
| license_number | string, nullable | required if type=individual_driver |
| business_reg_gstin | string, nullable | required if type=transport_company |
| vehicle_ids | UUID[] | FK -> VEHICLE |
| service_area | string | district/region |
| status | enum [pending, active, suspended, rejected] | |
| trust_profile_id | UUID FK -> TrustProfile | |

### Rules
```
ON logistics.register:
  IF type == "individual_driver": validate license_number format
  IF type == "transport_company": validate gstin format
  IF vehicle.registration_number already active under a different logistics_id:
    REJECT (409, "vehicle_already_registered")
  logistics.status = "pending"
  create_trust_profile(logistics, base_score=50)
  enqueue_admin_review(logistics)

-- suspension does not cancel in-progress shipments:
ON auto_suspend(logistics, reason="delivery_failure_threshold"):
  logistics.status = "suspended"
  -- existing active SHIPMENT rows continue to completion or manual admin reassignment
  -- only blocks NEW shipment assignment going forward
```

## Entity: ADMIN
Not self-registered. Provisioned directly via internal DB seed/migration script only. No public API endpoint may create an admin account under any condition — this must be enforced structurally (no route exists), not just by policy.

## Shared OTP rules
```
otp.expiry = 5 minutes
otp.max_resend_attempts = 3
ON exceeding max_resend_attempts: temporary_lockout(phone, duration=15 minutes)
```

## Auth
JWT-based. Access token TTL = 24h. Refresh token TTL = 30 days. Token payload includes `account_id` and `active_profile_id` (set after role-switch selection if multiple profiles exist on one phone).

## API Endpoints

| Method | Path | Body | Response |
|---|---|---|---|
| POST | /auth/otp/request | phone | 200 |
| POST | /auth/otp/verify | phone, otp | { accounts_linked: [...profile summaries] } |
| POST | /auth/select-role | account_id, profile_id | JWT |
| POST | /register/fpo | name, reg_number, address, ... | fpo object, 201 (status=pending) |
| POST | /register/farmer | name, aadhaar_number, gps_location, ... | farmer object, 201 (status=pending) |
| POST | /register/consumer | name | consumer object, 201 (status=active) |
| POST | /register/bulkbuyer | business_name, gstin, ... | bulkbuyer object, 201 (status=pending) |
| POST | /register/logistics | type, license_number\|gstin, vehicle_details | logistics object, 201 (status=pending) |
| POST | /admin/verify/:profile_type/:id | approve\|reject, reason? | profile object |

## Explicitly out of scope for MVP
- Real UIDAI Aadhaar e-KYC (requires AUA/KUA license) — format validation only
- Real MCA/GSTN database cross-verification — format validation + manual admin review only
- Credit-based payment terms for Bulk Buyers
