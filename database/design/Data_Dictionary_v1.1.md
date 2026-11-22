# Poker Game - Data Dictionary v1.1

**Project:** Multiplayer No-Limit Texas Hold'em Poker Game  
**Database:** PostgreSQL  
**Version:** 1.1   
**Reference:** Project_Schema_v1.1(2)

## 1. Overview

This document defines the database attributes, data types, constraints, and business meanings used in the Poker Game application.

The database consists of 10 tables:

1. USER
2. ROOM
3. ROOM_PARTICIPANT
4. SESSION
5. SESSION_PARTICIPANT
6. HAND
7. HAND_PLAYER
8. ACTION
9. CARDS
10. CHIP_TRANSACTION

All chips are virtual and exist only within a Session. The application does not maintain persistent user wallets or support cash-out.

### Conventions

| Notation | Meaning |
|---|---|
| PK | Primary Key |
| FK | Foreign Key |
| PK, FK | Attribute serves as both primary key and foreign key |
| Composite PK | Primary key consisting of multiple attributes |
| Optional | Attribute may contain NULL |
| Required | Attribute must contain a non-NULL value |
| TIMESTAMPTZ | PostgreSQL timestamp with time zone |
| BIGINT | 64-bit integer identifier |
| INTEGER | 32-bit integer |

Data types and enumerated values below are proposed implementation choices unless explicitly defined by the existing game rules.

---

## 2. USER

Stores registered user accounts and authentication information.

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| user_id | BIGINT | PK | Unique identifier for each user |
| username | VARCHAR(50) | Required | User's display name |
| email | VARCHAR(255) | Required, UNIQUE | Email address used for account identification |
| password_hash | TEXT | Required | Securely hashed user password |

**Business Notes**

- Passwords must never be stored in plaintext.
- Each email address identifies at most one account.
- A user may join multiple Rooms and participate in multiple Sessions.

---

## 3. ROOM

Stores information about poker rooms created by users.

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| room_id | BIGINT | PK | Unique room identifier |
| room_code | VARCHAR(6) | Required | Six-character room access code |
| password_hash | TEXT | Optional | Hashed room password, if password protection is enabled |
| status | VARCHAR(20) | Required | Current room lifecycle status |
| user_id | BIGINT | FK → USER | User who created the room |
| created_at | TIMESTAMPTZ | Required | Timestamp when the room is created |

**Proposed Status Values**

| Value | Description |
|---|---|
| WAITING | Room created and waiting for the Session to start |
| ACTIVE | Session is in progress |
| CLOSED | Room is no longer available |

**Business Notes**

- Each Room has exactly one Session.
- A Room can contain up to 20 current members, including spectators.
- A Room that has not started within 15 minutes of creation is automatically closed.
- The room access code must be unique among rooms that are currently joinable.

**Design Note:** The current relational schema uses `ROOM.user_id` to identify the creator. The earlier design used `created_by`; either name is valid, but naming should be consistent across documentation and implementation.

---

## 4. ROOM_PARTICIPANT

Tracks users who have joined a Room, including both seated players and spectators.

**Composite Primary Key:** `(room_id, user_id)`

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| room_id | BIGINT | PK, FK → ROOM | Room joined by the user |
| user_id | BIGINT | PK, FK → USER | User participating in the room |
| first_join_at | TIMESTAMPTZ | Required | Timestamp of the user's first entry into the room |
| last_leave_at | TIMESTAMPTZ | Optional | Timestamp of the user's most recent departure |
| status | VARCHAR(10) | Required | Current room membership status |

**Allowed Status Values**

| Value | Description |
|---|---|
| ACTIVE | User is currently a room member |
| LEFT | User voluntarily left the room |
| KICKED | User was removed by the room owner |

**Business Notes**

- A user can have only one RoomParticipant record per Room.
- A user with status LEFT may rejoin the same Room.
- A user with status KICKED cannot rejoin that Room.
- Standing up from a seat does not change RoomParticipant status.
- A spectator may exist in RoomParticipant without having a SessionParticipant record.

---

## 5. SESSION

Stores the lifecycle and configured duration of a poker session.

Each Room corresponds to exactly one Session.

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| session_id | BIGINT | PK | Unique Session identifier |
| room_id | BIGINT | FK → ROOM, UNIQUE | Room associated with the Session |
| duration | SMALLINT | Required | Configured Session duration in minutes |
| status | VARCHAR(20) | Required | Current Session lifecycle status |
| start_at | TIMESTAMPTZ | Optional | Actual Session start time |
| end_at | TIMESTAMPTZ | Optional | Actual Session completion time |

**Allowed Duration Values**

30, 60, 90, 120 minutes.

**Allowed Status Values**

| Value | Description |
|---|---|
| WAITING | Session exists but has not started |
| ACTIVE | Session is running |
| COMPLETED | Session finished normally |
| CANCELLED | Session was cancelled before completion |

**Business Notes**

- A WAITING Session is created when its Room is created.
- At least two players must be seated before the owner can start the Session.
- `start_at` remains NULL until the Session starts.
- `end_at` remains NULL until the Session finishes or is cancelled.
- When the configured duration expires, the current Hand is allowed to finish before the Session is completed.

---

## 6. SESSION_PARTICIPANT

Stores each player's Session-specific chip balance and current seating information.

A SessionParticipant record is created when a user first buys in.

**Composite Primary Key:** `(session_id, user_id)`

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| session_id | BIGINT | PK, FK → SESSION | Session in which the player participates |
| user_id | BIGINT | PK, FK → USER | Participating user |
| seat_no | SMALLINT | Optional | Current seat number, from 1 to 9 |
| stack_balance | INTEGER | Required | Chips currently available for the player to use |

**Business Notes**

- A user must complete an initial Buy-in before sitting down.
- The initial Buy-in amount must be between 100 and 400 chips.
- `seat_no = NULL` indicates that the player is not currently seated.
- A player who stands up retains their SessionParticipant record and chip balance.
- A player may sit down again without performing another initial Buy-in.
- `stack_balance` excludes chips already committed to the current pot.
- `stack_balance` must not be negative.
- Within a Session, each non-NULL seat number may be assigned to at most one player.
- At Session completion, the remaining stack is used to determine the player's final result.

**Important Constraints**

- `CHECK (seat_no BETWEEN 1 AND 9)`
- `CHECK (stack_balance >= 0)`
- Unique `(session_id, seat_no)` for non-NULL seat numbers.

---

## 7. HAND

Stores information about each poker Hand played during a Session.

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| hand_id | BIGINT | PK | Unique Hand identifier |
| session_id | BIGINT | FK → SESSION | Session containing the Hand |
| hand_no | INTEGER | Required | Sequential Hand number within the Session |
| dealer_seat | SMALLINT | Required | Dealer/button seat number for the Hand |
| start_at | TIMESTAMPTZ | Required | Timestamp when the Hand begins |
| end_at | TIMESTAMPTZ | Optional | Timestamp when the Hand finishes and is settled |

**Business Notes**

- Hand numbers are unique within a Session.
- The dealer seat must be between 1 and 9.
- `end_at = NULL` indicates a Hand that has not yet been marked complete.
- Each Hand may involve multiple seated players.
- The game engine determines the blinds, action order, and Hand outcome.

**Important Constraints**

- `UNIQUE (session_id, hand_no)`
- `CHECK (dealer_seat BETWEEN 1 AND 9)`
- `CHECK (end_at IS NULL OR end_at >= start_at)`

---

## 8. HAND_PLAYER

Records the participation and chip balance changes of a player in a specific Hand.

**Composite Primary Key:** `(hand_id, user_id)`

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| hand_id | BIGINT | PK, FK → HAND | Hand in which the player participated |
| user_id | BIGINT | PK, FK → USER | Participating player |
| seat_no | SMALLINT | Required | Player's seat number during this Hand |
| starting_stack | INTEGER | Required | Player's available chip balance at the beginning of the Hand, before blinds and bets |
| ending_stack | INTEGER | Optional | Player's available chip balance after the Hand is settled |

**Business Notes**

- Each player can appear only once in a Hand.
- A player must have a valid SessionParticipant record for the Hand's Session.
- `seat_no` preserves the historical seat assignment even if the player later changes seats.
- `ending_stack` remains NULL until the Hand is settled.
- Chip changes caused by the Hand are reflected in the difference between ending_stack and starting_stack, assuming no external chip adjustment occurs during the Hand.
- Each player may perform zero or more Actions during a Hand.
- Each normally dealt-in player receives two private cards.

**Important Constraints**

- `UNIQUE (hand_id, seat_no)`
- `CHECK (seat_no BETWEEN 1 AND 9)`
- `CHECK (starting_stack >= 0)`
- `CHECK (ending_stack IS NULL OR ending_stack >= 0)`

---

## 9. ACTION

Stores the sequence of player betting decisions within a Hand.

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| action_id | BIGINT | PK | Unique Action identifier |
| hand_id | BIGINT | Composite FK → HAND_PLAYER | Hand in which the Action occurred |
| user_id | BIGINT | Composite FK → HAND_PLAYER | Player performing the Action |
| action_no | INTEGER | Required | Sequential Action number within the Hand |
| street | VARCHAR(10) | Required | Betting round in which the Action occurred |
| action_type | VARCHAR(20) | Required | Type of player action |
| add_amount | INTEGER | Required | Additional chips committed by this Action |
| bet_to | INTEGER | Required | Player's cumulative committed amount in the current betting street after the Action |

**Allowed Street Values**

| Value | Description |
|---|---|
| PREFLOP | Betting round before community cards |
| FLOP | Betting round after the first three community cards |
| TURN | Betting round after the fourth community card |
| RIVER | Betting round after the fifth community card |

**Proposed Action Types**

| Value | Description |
|---|---|
| FOLD | Player forfeits the Hand |
| CHECK | Player passes without adding chips |
| CALL | Player matches the required bet, possibly with a short all-in |
| BET | Player opens betting on the current street |
| RAISE | Player increases the current bet |
| ALL_IN | Player commits all remaining available chips |

**Business Notes**

- Each Action must belong to a player participating in the corresponding Hand.
- `action_no` is unique within each Hand.
- `add_amount` is the amount newly committed by the current Action.
- `bet_to` is the player's total committed amount for the current street after that Action, not the total across the entire Hand.
- For CHECK and FOLD, `add_amount = 0`.
- The game engine validates legal actions, minimum raises, all-in rules, and action order.
- The game engine enforces the 30-second action timer.

**Important Constraints**

- `UNIQUE (hand_id, action_no)`
- `CHECK (add_amount >= 0)`
- `CHECK (bet_to >= 0)`
- Composite FK `(hand_id, user_id)` references `HAND_PLAYER(hand_id, user_id)`.

**Design Note:** Whether ALL_IN is stored as a separate action type or represented by BET, CALL, or RAISE with an all-in condition should be finalized before implementation.

---

## 10. CARDS

Stores private hole cards and community cards dealt during a Hand.

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| card_id | BIGINT | PK | Unique card record identifier |
| hand_id | BIGINT | FK → HAND | Hand in which the card was dealt |
| user_id | BIGINT | Optional | Player receiving the card; NULL for community cards |
| card_code | VARCHAR(3) | Required | Card rank and suit identifier |
| card_position | SMALLINT | Required | Card position within the relevant card group |
| street | VARCHAR(10) | Required | Stage at which the card is dealt |

**Card Code Format**

Use rank followed by suit.

Examples:

- `AS` — Ace of Spades
- `KH` — King of Hearts
- `TD` — Ten of Diamonds
- `2C` — Two of Clubs

Allowed ranks: `2–9`, `T`, `J`, `Q`, `K`, `A`.

Allowed suits: `S`, `H`, `D`, `C`.

**Card Ownership**

| Card Type | user_id | card_position | street |
|---|---|---|---|
| Hole Card | Player ID | 1–2 | PREFLOP |
| Community Card | NULL | 1–5 | FLOP, TURN, RIVER |

For community cards, positions 1–3 correspond to the Flop, position 4 to the Turn, and position 5 to the River.

**Business Notes**

- Each physical card can appear only once within the same Hand.
- Each normally dealt-in player receives exactly two private cards.
- A completed Hand can have zero, three, four, or five community cards, depending on when betting ends.
- Community cards are shared by all players in the Hand.
- Private cards must not be disclosed to unauthorized users.

**Important Constraints**

- `UNIQUE (hand_id, card_code)`
- `CHECK (card_position BETWEEN 1 AND 5)`
- Composite FK `(hand_id, user_id)` references `HAND_PLAYER(hand_id, user_id)` when user_id is non-NULL.

**Implementation Note:** PostgreSQL can enforce the composite foreign key while keeping `hand_id` required and `user_id` optional using the default `MATCH SIMPLE` behavior. This constraint will be added in SQL if ERDPlus cannot represent the mixed nullability.

---

## 11. CHIP_TRANSACTION

Records Session-specific chip purchases and adjustments.

| Attribute | Data Type | Constraint | Description |
|---|---|---|---|
| transaction_id | BIGINT | PK | Unique chip transaction identifier |
| session_id | BIGINT | Composite FK → SESSION_PARTICIPANT | Session associated with the transaction |
| user_id | BIGINT | Composite FK → SESSION_PARTICIPANT | Player associated with the transaction |
| transaction_type | VARCHAR(20) | Required | Type of chip transaction |
| amount | INTEGER | Required | Number of chips involved in the transaction |
| created_at | TIMESTAMPTZ | Required | Timestamp when the transaction was recorded |

**Proposed Transaction Types**

| Value | Description |
|---|---|
| BUYIN | Initial purchase of Session chips |
| TOPUP | Additional chips purchased during the Session |
| ADJUST | Administrative correction to a player's chip balance |

**Business Notes**

- A player's first BUYIN creates their SessionParticipant record.
- The initial BUYIN must be between 100 and 400 chips.
- A player can have multiple chip transactions within the same Session.
- Each transaction must reference an existing SessionParticipant.
- Chip transactions do not record individual poker bets or pot winnings.
- Initial Buy-in and subsequent chip purchases increase the player's available stack.
- The Session's total chip purchases can be calculated from the transaction ledger.

**Implementation Notes**

- The sign convention for ADJUST transactions must be defined before applying an amount constraint.
- A request idempotency key may be added to prevent duplicate chip purchases caused by network retries.
- Creating the initial SessionParticipant record, recording BUYIN, and updating the chip balance must occur in one database transaction.

---

## 12. Cross-Table Integrity Rules

The following constraints apply across multiple tables.

| Rule | Enforcement |
|---|---|
| One Session per Room | UNIQUE constraint on SESSION.room_id; atomic Room/Session creation |
| One RoomParticipant per User per Room | Composite primary key |
| One SessionParticipant per User per Session | Composite primary key |
| One player per occupied seat | Partial unique index on session_id and seat_no |
| One Hand number per Session | Composite unique constraint |
| One HandPlayer per User per Hand | Composite primary key |
| One player per seat within a Hand | Composite unique constraint |
| One Action number per Hand | Composite unique constraint |
| Action must belong to a HandPlayer | Composite foreign key |
| Private Card must belong to a HandPlayer | Composite foreign key |
| ChipTransaction must belong to a SessionParticipant | Composite foreign key |
| Player must belong to the Hand's Session | Java validation in Phase 1 |
| At least two seated players before Session starts | Java validation |
| Maximum 20 current Room members | Java validation |
| No negative available chip balance | Database CHECK and game engine |
| No duplicate cards within a Hand | Composite unique constraint |
| Valid poker betting sequence | Java game engine |
| Session-specific chips only | Application and database design |

---

## 13. Derived Values

The following values can be calculated from existing data and do not require additional stored attributes in the current schema.

| Derived Value | Calculation |
|---|---|
| Hand net result | ending_stack − starting_stack |
| Total Session buy-in | Sum of applicable chip purchase transactions |
| Final Session stack | stack_balance at Session completion |
| Session net result | Final Session stack − total chips purchased, adjusted for any corrections |
| Hand duration | end_at − start_at |
| Session elapsed time | Current time or end_at minus start_at |
| Player seated status | seat_no IS NOT NULL |

Derived values should be calculated only when their required data is available and their underlying business assumptions hold.

---

## 14. Items to Finalize Before PostgreSQL DDL

The following decisions are not fully represented by the current relational schema and should be confirmed before implementation:

1. Restore `ROOM.created_at` or define another reliable persisted timestamp for the 15-minute room expiration rule.
2. Finalize Room and Session lifecycle status values and transitions.
3. Confirm the semantics of ALL_IN in ACTION.
4. Confirm whether ADJUST transactions may have negative amounts.
5. Determine how to prevent duplicate initial BUYIN transactions.
6. Decide whether HAND_PLAYER-to-SESSION_PARTICIPANT membership should be enforced by a database composite foreign key or Java validation.
7. Add the composite foreign key for CARDS in PostgreSQL DDL.
8. Decide whether the initial deck order or shuffle seed needs to be persisted for crash recovery and deterministic Hand replay.

**End of Data Dictionary v1.1**