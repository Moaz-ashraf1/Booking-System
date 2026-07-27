# 🏨 Booking System 

A full-scale, multi-tenant travel reservation platform for **hotels and flights**, sourced from external third-party providers in real time. Built as a team mentorship project, with each engineer owning a distinct service domain.

---

## 🌍 Project Overview

The platform connects three system actors to deliver an end-to-end booking experience:

- **Guest User** — unauthenticated browser exploring destinations and live rates, tracked via a session-based `guestToken`.
- **Registered User** — authenticated user with persistent search history, saved preferences, and profile/booking management.
- **System / Third-Party API Providers** — external hotel and flight suppliers queried dynamically via asynchronous request aggregation.

---

### Architecture at a Glance

```text
       ┌─────────────────────────────────────────────────────────────┐
       │                Core Hotel & Flight System                   │
       └──────────────────────────────┬──────────────────────────────┘
                                       │
 ┌──────────────────┬─────────────────┼─────────────────┬──────────────────┐
 │                   │                 │                 │                  │
 ▼                   ▼                 ▼                 ▼                  ▼
[Auth & Identity]  [Search & Filter] [Booking Engine]  [Payments]    [Notifications]
 (Keycloak/OAuth)     (My Scope)    (Hotels/Flights) (3rd-Party)   (Event-Driven)
```

---

### The 7 Service Domains

1. **User Registration & Authentication** — identity management via Keycloak, supporting Social Login (Google/Facebook), SSO, and OAuth 2.0 / OpenID Connect.
2. **Search & Filter Engine** — high-throughput search aggregator using a Scatter & Gather pattern against external provider APIs, with dual-level Redis/Postgres caching. *(My scope — see below)*
3. **Booking Management** — multi-entity reservation engine for hotel/flight holds, cancellations, and history tracking.
4. **Payment Integration** — asynchronous payment gateways with status verification and receipt generation.
5. **Notification System** — event-driven email/SMS confirmations and promotional pushes.
6. **Customer Management** — user preferences, addresses, reviews, and support.
7. **Dashboard & Reporting** — system-wide transactional metrics and analytics.

---

# 🎯 Use Cases

## Use Case 1: Destination & Hotel Autocomplete

**Actors:** Guest/User (Primary), Database (Secondary)

**Main Flow (Mermaid Flowchart):**

```mermaid
flowchart TD
    Start([Guest types in Destination field]) --> Debounce[Frontend debounces input - 300ms]
    Debounce --> EmptyCheck{Is query empty?}
    EmptyCheck -- Yes --> ShowRecent[Show recent searches by guestToken, or placeholder]
    ShowRecent --> End1([End])
    EmptyCheck -- No --> SendQuery[Frontend sends query to Backend]
    SendQuery --> CacheCheck{Cache hit?}
    CacheCheck -- Yes --> UseCache[Use cached results]
    CacheCheck -- No --> Prefix[Run Prefix Search - B-Tree - on Destination.name AND Hotel.name]
    Prefix --> PrefixFound{Any results found?}
    PrefixFound -- Yes --> UseResults[Use these results]
    PrefixFound -- No --> Fuzzy[Run Fuzzy Search - GIN pg_trgm - fallback on both tables]
    Fuzzy --> FuzzyFound{Any results found?}
    FuzzyFound -- No --> NoResults([Return empty results - End])
    FuzzyFound -- Yes --> UseResults
    UseResults --> Merge[Merge results from both tables, tag each with type: destination or hotel]
    UseCache --> Merge
    Merge --> StoreCache[Store merged results in Cache]
    StoreCache --> Return[Return results to Frontend]
    Return --> Display[Frontend displays suggestions list]
    Display --> End2([End: Success])
```

**Pseudocode:**

```
FUNCTION autocomplete(query, guestToken):

    IF query IS EMPTY:
        recentSearches = Database.getRecentSearches(guestToken, limit=5)
        RETURN recentSearches OR show_placeholder()

    cacheKey = "autocomplete:" + query
    cachedResult = Cache.get(cacheKey)
    IF cachedResult EXISTS:
        RETURN cachedResult

    destinationResults = Database.query(
        "SELECT * FROM Destination WHERE name LIKE query + '%' LIMIT 10"
    )
    hotelResults = Database.query(
        "SELECT * FROM Hotel WHERE name LIKE query + '%' LIMIT 10"
    )

    IF destinationResults IS EMPTY AND hotelResults IS EMPTY:
        destinationResults = Database.query(
            "SELECT * FROM Destination WHERE name % query LIMIT 10"
        )
        hotelResults = Database.query(
            "SELECT * FROM Hotel WHERE name % query LIMIT 10"
        )

    IF destinationResults IS EMPTY AND hotelResults IS EMPTY:
        RETURN empty_result()

    mergedResults = []
    FOR each d IN destinationResults:
        mergedResults.ADD({ type: "destination", id: d.id, name: d.name })
    FOR each h IN hotelResults:
        mergedResults.ADD({ type: "hotel", id: h.id, name: h.name, destinationId: h.destinationId })

    Cache.set(cacheKey, mergedResults, ttl = SHORT_TTL)
    RETURN mergedResults
```

---

## Use Case 2: Search Hotel

**Actors:** Guest/User (Primary), Hotel API Providers — N providers (Secondary)

**Preconditions:**

1. User does not need to be logged in (Guest access via guestToken)
2. System is successfully connected to at least one Hotel API Provider
3. User has selected a valid Destination from Autocomplete results
4. User has entered check-in/check-out dates and specified occupancy

**Business Rules:**

- If no check-out date is set → default to check-in date + 1 day
- Default: 1 Adult, 1 Room if occupancy is not specified
- At least 1 adult required per room
- "Search" button is disabled until Destination + check-in date are set

**Main Flow (Mermaid Flowchart):**

```mermaid
flowchart TD
    Start([Guest enters the system]) --> Fill[Guest fills search form: Destination via Autocomplete, Dates, Occupancy]
    Fill --> Click[Guest clicks Search button]
    Click --> Valid{Is input valid?}
    Valid -- No --> ErrEnd([Validation error - End])
    Valid -- Yes --> BuildKey[System builds Cache Key: destinationId + checkIn + checkOut + occupancy]
    BuildKey --> CacheCheck{Valid entry in Cache within TTL?}
    CacheCheck -- Yes --> UseCache[Return cached result]
    CacheCheck -- No --> Scatter[System scatters request to all Hotel API Providers in parallel]
    Scatter --> Gather[System gathers responses within timeout window]
    Gather --> AnyResponse{At least one provider responded?}
    AnyResponse -- No --> TechErr([Technical error - End])
    AnyResponse -- Yes --> Group[System groups results by giataId - same physical hotel]
    Group --> Select[System selects lowest price per group, keeps winningProvider]
    Select --> ExistsCheck{Hotel exists in DB by giataId?}
    ExistsCheck -- Yes --> SkipWrite[Skip DB write]
    ExistsCheck -- No --> StoreStatic[Store static data: name, photos, location, giataId, provider ID mapping - no price]
    SkipWrite --> StoreCache[Store merged result - price + winningProvider - in Cache, short TTL]
    StoreStatic --> StoreCache
    StoreCache --> ResultsCheck
    UseCache --> ResultsCheck{Any results?}
    ResultsCheck -- No --> NoRes([No hotels available - End])
    ResultsCheck -- Yes --> Display[System displays Hotel Cards to user]
    Display --> Success([End: Success])
```

---

**Sequence Diagram — High-level (Mermaid):**

```mermaid
sequenceDiagram
    participant G as Guest
    participant F as Frontend
    participant B as Backend
    participant C as Cache

    G->>F: fill form + click Search
    F->>F: validate input
    alt invalid input
        F-->>G: show validation error
    else valid input
        F->>B: send search request
        B->>C: check cache key
        alt cache hit
            C-->>B: return cached result
        else cache miss
            Note over B: Fetch & Merge Live Prices (see Zoom-in diagram)
            alt all providers failed
                B-->>F: technical error
                F-->>G: show error message
            else at least one responded
                B->>B: merged result ready
            end
        end
        alt no results
            B-->>F: no results
            F-->>G: show "No hotels available"
        else results found
            B-->>F: return final hotel list
            F-->>G: display hotel cards
        end
    end
```

---

**Sequence Diagram — Zoom-in: Fetch Live Prices (Mermaid):**

```mermaid
sequenceDiagram
    participant B as Backend
    participant P as Providers (1..N)
    participant D as Database
    participant C as Cache

    Note over B: Cache miss (from high-level flow)
    B->>P: scatter request (destination, dates, occupancy) in parallel
    Note over B,P: wait for responses within fixed timeout window

    alt no provider responded within timeout
        B-->>B: mark as failure
        Note over B: return error to Frontend
    else at least one provider responded
        P-->>B: gather partial/full responses
        B->>B: group results by giataId
        B->>B: select lowest price per group, keep winningProvider
        B->>D: check if Hotel exists (match by giataId)
        alt hotel already exists
            D-->>B: exists, skip write
        else hotel does not exist
            B->>D: store static hotel data (name, photo, star, giataId, provider mapping) - no price
            D-->>B: stored
        end
        B->>C: store merged result (price + winningProvider + static data), short TTL
        B-->>B: return final merged result
    end
```

**Pseudocode:**

```
FUNCTION searchHotel(destinationId, checkIn, checkOut, occupancy):

    IF destinationId IS NULL OR checkIn IS NULL:
        RETURN error("Please select a destination and check-in date")

    IF checkOut IS NULL:
        checkOut = checkIn + 1 DAY

    IF occupancy IS NULL:
        occupancy = { adults: 1, rooms: 1 }

    cacheKey = buildCacheKey(destinationId, checkIn, checkOut, occupancy)
    cachedResult = Cache.get(cacheKey)
    IF cachedResult EXISTS:
        RETURN cachedResult

    providerResponses = []
    PARALLEL FOR each provider IN HotelProviders:
        TRY:
            response = provider.fetchHotels(destinationId, checkIn, checkOut, occupancy)
            providerResponses.ADD(response)
        CATCH (timeout OR error):
            CONTINUE
        WAIT UP TO timeoutLimit

    IF providerResponses IS EMPTY:
        RETURN error("Unable to fetch hotels right now, please try again")

    groupedByGiataId = {}
    FOR each response IN providerResponses:
        FOR each hotelOffer IN response.hotels:
            groupedByGiataId[hotelOffer.giataId].ADD(hotelOffer)

    finalResults = []
    FOR each (giataId, offers) IN groupedByGiataId:
        bestOffer = offers.MIN_BY(price)

        existingHotel = Database.findByGiataId(giataId)
        IF existingHotel IS NULL:
            newHotel = Database.insert(Hotel, {
                name: bestOffer.name, description: bestOffer.description,
                destinationId: destinationId, starRating: bestOffer.starRating,
                lat: bestOffer.lat, lng: bestOffer.lng, giataId: giataId
            })
            Database.insert(HotelProviderMapping, {
                hotelId: newHotel.id, providerName: bestOffer.providerName,
                providerHotelId: bestOffer.providerHotelId
            })
            hotelData = newHotel
        ELSE:
            hotelData = existingHotel

        finalResults.ADD({
            hotel: hotelData, price: bestOffer.price,
            winningProvider: bestOffer.providerName
        })

    IF finalResults IS EMPTY:
        RETURN noResultsMessage()

    Cache.set(cacheKey, finalResults, ttl = SHORT_TTL)
    RETURN finalResults
```

---

# Entity Relationships (Final Version)

```
Destination: id (PK), name (+Trigram Index), type, parentId (FK self, nullable), createdAt, updatedAt

Hotel: id (PK), name (+Trigram Index), description, destinationId (FK), starRating, lat, lng, giataId (UNIQUE)

HotelPhoto: id (PK), hotelId (FK), url, isPrimary

RoomType: id (PK), hotelId (FK), name, description, numberOfBeds, capacity

RoomTypePhoto: id (PK), roomTypeId (FK), url, isPrimary

HotelProviderMapping: id (PK), hotelId (FK), providerName, providerHotelId
```

**ER Diagram (Mermaid):**

```mermaid
erDiagram
    DESTINATION ||--o{ HOTEL : has
    HOTEL ||--o{ HOTELPHOTO : has
    HOTEL ||--o{ ROOMTYPE : has
    HOTEL ||--o{ HOTELPROVIDERMAPPING : has
    ROOMTYPE ||--o{ ROOMTYPEPHOTO : has

    DESTINATION {
        uuid id PK
        string name
        string type
        uuid parentId FK
        timestamp createdAt
        timestamp updatedAt
    }
    HOTEL {
        uuid id PK
        string name
        string description
        uuid destinationId FK
        int starRating
        decimal lat
        decimal lng
        string giataId
    }
    HOTELPHOTO {
        uuid id PK
        uuid hotelId FK
        string url
        boolean isPrimary
    }
    ROOMTYPE {
        uuid id PK
        uuid hotelId FK
        string name
        string description
        int numberOfBeds
        int capacity
    }
    ROOMTYPEPHOTO {
        uuid id PK
        uuid roomTypeId FK
        string url
        boolean isPrimary
    }
    HOTELPROVIDERMAPPING {
        uuid id PK
        uuid hotelId FK
        string providerName
        string providerHotelId
    }
```

---

# Guest Token → User Account Migration

```
1. First site visit → Backend generates a guestToken, stores it in a Cookie
2. Any activity (search, hold) done as guest → recorded as (guestToken, userId = NULL)
3. User signs up / logs in → browser automatically attaches the guestToken
   with the same Login Request
4. Backend validates login credentials, retrieves the userId
5. Backend finds the guestToken attached to the request → runs an UPDATE
   on all records (userId = NULL, guestToken = "xyz") → (userId = [new account])
6. From this point forward, any new activity is recorded directly under userId
```
