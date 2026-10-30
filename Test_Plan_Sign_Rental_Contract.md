# BeAvis Car Rental System: Design Specification and Test Plan

**Course:** CS 250, Fall 2026
**Team:** Alec, Vunsh, Gabe, Meshach, Phong (Group 08)
**Author of this section:** Phong Tran
**Date:** October 16, 2026

---

## 1. Software Design Specification

The architecture is unchanged from Assignment 2. The class diagram has one addition: `Vehicle` gains a `dailyRate: Double` attribute.

### Software Architecture Overview

```text
[ Mobile App / Web App ]             [ Employee Interface ]
         |                                      |
         +--------------- HTTPS / REST ---------+
                            |
                  [ API Gateway / Router ]
                            |
             +--------------+--------------+
             |              |              |
          [Auth]          [Fleet]       [Rentals]
             |              |              |
             +--------------+--------------+
                            |
                    [ Data Access Layer ]
                            |
                [ PostgreSQL Database ]
                            |
                [ Encrypted PDF Storage ]
```

The mobile app, web app, and employee interface send requests to the API Gateway using HTTPS and REST. The API Gateway routes requests to the correct backend service. Auth manages account sign-in and authentication, Fleet manages vehicle records and availability, and Rentals manages reservations and rental contracts. These services use the data access layer to read and update information in the PostgreSQL database. Signed rental PDFs are kept in encrypted document storage, while the database stores the related reservation and document references. Passwords are stored as hashes, and payment details are represented by a payment token rather than a full card number.

### UML Class Diagram

```text
                          +---------------------------------------+
                          |                  User                 |
                          +---------------------------------------+
                          | - userID: String                      |
                          | - email: String                       |
                          | - passwordHash: String                |
                          | - is2FAEnabled: Boolean               |
                          +---------------------------------------+
                          | + login(email, password): Boolean     |
                          | + verifyEmail(code): Boolean          |
                          | + enable2FA(secret): Boolean          |
                          +---------------------------------------+
                                              ^
                                              | (inheritance)
                   +--------------------------+------------------------+
                   |                                                   |
+--------------------------------------+              +----------------------------------+
|               Employee               |              |             Customer             |
+--------------------------------------+              +----------------------------------+
| - employeeID: String                 |              | - phoneNumber: String            |
| - departmentRole: String             |              +----------------------------------+
| - currentTask: String                |              | + searchNearbyLocations(         |
+--------------------------------------+              |     lat, long): List<Location>   |
| + lookUpCustomer(query): Customer    |              +----------------------------------+
| + reviewContract(resID): Contract    |                      | 1                   | 1
| + updateVehicleStatus(               |                      | saves               | makes
|     vehicleID, status): Boolean      |                      | 0..*                | 0..*
+--------------------------------------+                      |                     |
+-----------------------------------+                 +------------------------------------+
|           PaymentMethod           |                 |         RentalReservation          |
+-----------------------------------+                 +------------------------------------+
| - paymentID: String               |                 | - reservationID: String            |
| - customerID: String              |                 | - customerID: String               |
| - type: PaymentType               | 1          0..* | - vehicleID: String                |
| - paymentToken: String            |-----------------| - paymentID: String                |
| - billingPostalCode: String       |    pays for     | - startDate: Date                  |
+-----------------------------------+                 | - endDate: Date                    |
| + validateToken(): Boolean        |                 | - totalCost: Double                |
| + updateBillingAddress(           |                 | - startCondition: String           |
|     postalCode): Boolean          |                 | - endCondition: String             |
+-----------------------------------+                 | - status: ReservationStatus        |
                                                      +------------------------------------+
+--------------------------------------+              | + calculateTotal(rate, days):      |
|               Contract               |              |     Double                         |
+--------------------------------------+              | + signContract(signature): Boolean |
| - contractID: String                 | 1          1 | + recordCondition(damage,          |
| - reservationID: String              |--------------|     isStartCondition): Boolean     |
| - documentURI: String                |    covers    | + cancel(): Boolean                |
| - signedAt: Date                     |              +------------------------------------+
+--------------------------------------+                                            | 0..*
| + generate(reservationID): Boolean   |                                            | reserves
| + recordSignature(signature):        |                                            | 1
|     Boolean                          |              +------------------------------+
| + retrieveDocument(): String         |              |           Vehicle            |
+--------------------------------------+              +------------------------------+
                                                      | - vehicleID: String          |
+-----------------------------------+                 | - locationID: String         |
|              Location             |                 | - make: String               |
+-----------------------------------+                 | - model: String              |
| - locationID: String              |                 | - dailyRate: Double   (NEW)  |
| - name: String                    | 1          0..* | - status: VehicleStatus      |
| - address: String                 |-----------------+------------------------------+
| - latitude: Double                |       has       | + setMaintenance(status,     |
| - longitude: Double               |                 |     notes): Boolean          |
+-----------------------------------+                 | + isAvailable(start, end):   |
| + getAvailableVehicles(start,     |                 |     Boolean                  |
|     end): List<Vehicle>           |                 +------------------------------+
| + getDirections(fromLat,          |
|     fromLong): String             |
+-----------------------------------+
```

User is the parent class for Customer and Employee, so both share the account fields and login operations. A Customer can make multiple RentalReservations, and each reservation belongs to one customer. Each reservation is for one Vehicle; the same vehicle may be used in different reservations over time, but confirmed reservations cannot overlap. A reservation uses one PaymentMethod, while a customer may have more than one saved payment method. Each reservation has a Contract containing the rental agreement and a reference to its stored document.

### Changes from Assignment 2

| Change | Reason |
| --- | --- |
| Added `dailyRate: Double` to `Vehicle` | `RentalReservation.calculateTotal(rate, days)` needs a daily rate, but no class stored one |
| `lookUpCustomer()` listed under `Employee` only | Assignment 2 text had it under Customer while the diagram had it under Employee |
| `Customer.searchNearbyLocations()` is the official name | Text said `searchNearbyVehicles`; the diagram name was kept |
| `PaymentMethod.paymentToken` is the official name | Text said `cardToken`; the diagram name was kept |

Reservation status rule used by the tests: a reservation is created as `PENDING` and becomes `CONFIRMED` when the contract is signed.

### Test granularities

Each test set covers one feature at three granularities. **Unit** tests check one method with everything else stubbed. **Integration** tests check two or more classes working together through their real interfaces. **System** tests run the full feature through the app the way a user would.

---

## 2. Test Set: Sign Rental Contract

**Feature:** The customer signs the contract for a pending reservation, the signed PDF is stored, and the reservation becomes CONFIRMED. An employee can then review it.

**Scope:** `Contract`, `RentalReservation`, `Employee`, encrypted PDF storage (Rentals service).

```text
[ Mobile App ]   [ Employee Interface ]   <-- System test covers both clients
      |                   |
      +----[ API Gateway ]+
                 |
             [Rentals]                    <-- Integration test: RentalReservation + Contract + storage
                 |
        [ Data Access Layer ]
           |            |
    [ PostgreSQL ]  [ Encrypted PDF Storage ]

                                              Unit test covers Contract.recordSignature() only
```

### Unit test: `Contract.recordSignature(signature)`

Fixture: contract C500 for reservation R200, `signedAt = null`. Storage is stubbed.

| ID | Starting state | Input | Expected return | Expected `signedAt` | Covers |
| --- | --- | --- | --- | --- | --- |
| U1 | unsigned | valid signature bytes | true | set to now | valid input |
| U2 | unsigned | empty array | false | still null | invalid input |
| U3 | already signed | valid signature bytes | false | unchanged | no double signing |

Covers the valid partition, the invalid partition, and the rule that a signed contract cannot be overwritten.

### Integration test: `RentalReservation.signContract()` with `Contract` and PDF storage

Fixture: reservation R200 with status PENDING and a generated contract, using a test database and test storage bucket.

| ID | Setup | Action | Return | `status` after | Storage |
| --- | --- | --- | --- | --- | --- |
| I1 | as fixture | `signContract(valid)` | true | CONFIRMED | one PDF written, `documentURI` points to it |
| I2 | as fixture | `signContract(empty)` | false | PENDING | nothing written |
| I3 | storage write forced to fail | `signContract(valid)` | false | PENDING | `signedAt` still null |

I1 shows the three components work together. I2 shows the reservation respects Contract's rejection. I3 is the key integration case: if storage fails after the signature is recorded, the reservation must not end up CONFIRMED with no document.

### System test: customer signs, employee reviews

Preconditions: reservation R200 is PENDING for the test customer; a test employee account exists

| Step | Actor | Action | Expected |
| --- | --- | --- | --- |
| 1 | Customer | Open R200 in "My Rentals", tap "Sign contract" | PDF shows correct car, dates, and $135 total |
| 2 | Customer | Submit with no signature | "Signature required"; status still PENDING |
| 3 | Customer | Draw signature, submit | Success; status now CONFIRMED |
| 4 | Employee | Search R200 with `reviewContract`, open document | Signed PDF opens with the signature and signed timestamp |
| 5 | Other customer | Request R200's document link directly | Access denied |

Steps 1 to 3 cover the customer requirement including the negative case, step 4 covers the employee requirement, step 5 covers the authorization rule in `retrieveDocument()`.

---

## 3. Summary

| Feature | Unit | Integration | System |
| --- | --- | --- | --- |
| Sign Rental Contract | `Contract.recordSignature()` | `RentalReservation` + `Contract` + storage | customer signs, employee reviews |

The Make a Rental test set is written separately by Alec.
