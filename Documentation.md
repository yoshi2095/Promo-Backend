# Promo Management Service - Knowledge Transfer Document


**For:** Frontend Engineers New to Backend Development 
**Author:** Knowledge Transfer Guide 
**Date:** February 2026


---


## Table of Contents


1. [Project Overview](#project-overview)
2. [Campaign Creation Flow - Deep Dive](#campaign-creation-flow)
3. [Campaign Update Flow](#campaign-update-flow)
4. [Additional Information](#additional-information)


---


## Project Overview


### 1. How to Run This Project Locally


#### Prerequisites
- **Java 17** (JDK 17)
- **Maven 3.6+** (for building and dependency management)
- **MySQL 8.0+** (running on localhost:3306)
- **MongoDB** (running on localhost:27017)
- **Redis** (running on localhost:6379)
- **Kafka** (running on localhost:9092) - Optional for local development


#### Step-by-Step Setup


1. **Clone and Navigate**
  ```bash
  cd promo-management-service
  ```


2. **Configure Database Connections**
  - Edit `promo-management/src/main/resources/application.yml`
  - Update MySQL connection (lines 145-147):
    ```yaml
    spring:
      datasource:
        url: jdbc:mysql://localhost:3306/nextgen_promo?allowPublicKeyRetrieval=true&useSSL=false
        username: root
        password: password  # Change to your MySQL password
    ```
  - MongoDB connection (lines 152-153):
    ```yaml
    spring:
      data:
        mongodb:
          uri: mongodb://localhost:27017/nextgen_promo_dev1
    ```
  - Redis connection (lines 149-151):
    ```yaml
    spring:
      data:
        redis:
          host: localhost
          port: 6379
    ```


3. **Create MySQL Database**
  ```sql
  CREATE DATABASE nextgen_promo;
  ```
  Liquibase will automatically create tables on first run.


4. **Build the Project**
  ```bash
  mvn clean install
  ```
  This compiles the code, runs tests, and packages the application.


5. **Run the Application**
  ```bash
  cd promo-management
  mvn spring-boot:run
  ```
  Or run the main class `PromoManagementApplication` from your IDE (IntelliJ IDEA, Eclipse, VS Code).


6. **Verify It's Running**
  - Application starts on port **8022**
  - Base URL: `http://localhost:8022/api/promo-management`
  - Swagger UI: `http://localhost:8022/api/promo-management/swagger-ui.html`
  - Health Check: `http://localhost:8022/api/promo-management/actuator/health`


#### IDE Configuration (IntelliJ IDEA)


1. **Import Project**
  - File → Open → Select `promo-management-service` folder
  - Choose "Import project from external model" → Maven
  - Click "Next" until done


2. **Set JDK**
  - File → Project Structure → Project Settings → Project
  - Set SDK to Java 17


3. **Run Configuration**
  - Run → Edit Configurations
  - Add new Spring Boot configuration
  - Main class: `com.jubl.food.promo.management.PromoManagementApplication`
  - Working directory: `$MODULE_DIR$/promo-management`


---


### 2. Dependencies Overview


#### Databases
- **MySQL 8.0** (`mysql-connector-java`)
 - Used for: Relational data (campaign groups, metadata, static data)
 - Managed by: **Liquibase** (database migration tool)
 - Location: `promo-management-db/src/main/resources/liquibase/`


- **MongoDB**
 - Used for: Campaign documents (drafts, published campaigns, history)
 - Collections: `campaign_draft`, `campaign`, `campaign_history`, `campaign_summary`
 - Why MongoDB? Campaigns are complex nested JSON documents - MongoDB fits perfectly


#### Caching
- **Redis** (`spring-boot-starter-data-redis`)
 - Used for: Query result caching, session management
 - Configuration: `application.yml` lines 125-131


- **Caffeine** (In-memory cache)
 - Used for: Tenant configurations, frequently accessed data
 - Configuration: `application.yml` lines 178-192


#### Message Queue
- **Apache Kafka** (`spring-kafka`)
 - Used for: Asynchronous event processing
 - Topics:
   - `personalized-coupon-file-event` - File upload events
   - `personalised-coupon-topic` - Personal coupon generation
   - `open-coupon-topic` - Shareable coupon generation
   - `promo-sms-notification` - SMS notifications
   - `promo-email-notification` - Email notifications
 - Configuration: `application.yml` lines 62-124


#### External Microservices (REST Clients)
Located in `promo-management/src/main/java/com/jubl/food/promo/management/clients/`:


- **AdminClient** (`AdminClient.java`)
 - Base URL: `http://localhost:8083/nxg-admin/`
 - Purpose: Fetch tenant information, configurations
 - Uses: `RestTemplate` with retry mechanism


- **CalculatorMetadataClient** (`CalculatorMetadataClient.java`)
 - Base URL: `http://localhost:8085/`
 - Purpose: Fetch metadata attributes and values for campaign rules


- **NxgNotificationClient** (`NxgNotificationClient.java`)
 - Base URL: Configured in `application.yml`
 - Purpose: Send SMS/Email notifications


- **RedemptionQueryClient** (`RedemptionQueryClient.java`)
 - Base URL: `http://localhost:9091/`
 - Purpose: Query campaign redemption data


#### Other Key Dependencies
- **Spring Boot 3.1.2** - Main framework
- **Spring Data MongoDB** - MongoDB integration
- **Spring Data JPA** - MySQL integration
- **Lombok** - Reduces boilerplate code (getters, setters, constructors)
- **MapStruct** - DTO to Entity mapping
- **QueryDSL** - Type-safe queries
- **Jackson** - JSON serialization/deserialization


---


### 3. Project Structure Explained


```
promo-management-service/
├── promo-management/              # Main application module
│   ├── src/main/java/
│   │   └── com/jubl/food/promo/management/
│   │       ├── PromoManagementApplication.java  # Main entry point
│   │       ├── campaigns/                        # Campaign domain
│   │       │   ├── drafts/                      # Draft campaign logic
│   │       │   │   ├── controllers/            # REST endpoints for drafts
│   │       │   │   ├── CampaignDraftService.java      # Interface
│   │       │   │   └── CampaignDraftServiceImpl.java # Implementation
│   │       │   ├── published/                   # Published campaign logic
│   │       │   │   ├── controllers/
│   │       │   │   ├── CampaignService.java
│   │       │   │   └── CampaignServiceImpl.java
│   │       │   ├── summary/                    # Campaign summary logic
│   │       │   └── toggle/                     # Campaign activation/deactivation
│   │       ├── common/models/campaigns/        # DTOs (Data Transfer Objects)
│   │       │   ├── CampaignDTO.java            # Main campaign DTO
│   │       │   ├── CampaignVersionDTO.java     # Versioned campaign DTO
│   │       │   └── CampaignDetailDTO.java      # Campaign details DTO
│   │       ├── clients/                         # External service clients
│   │       ├── configs/                         # Configuration classes
│   │       ├── utils/                          # Utility classes
│   │       └── valueGroups/                    # Value group integration
│   └── src/main/resources/
│       └── application.yml                      # Main configuration
│
└── promo-management-db/          # Database module
   └── src/main/resources/
       └── liquibase/                           # Database migrations
```


#### Layer Architecture (Spring MVC Pattern)


**1. Controller Layer** (`*Controller.java`)
- **Purpose**: Handles HTTP requests/responses
- **Responsibilities**:
 - Receive HTTP requests
 - Validate input (using `@Validated` and `@JsonView`)
 - Call service layer
 - Return HTTP responses
- **Example**: `CampaignDraftController.java`


**2. Service Layer** (`*Service.java` and `*ServiceImpl.java`)
- **Purpose**: Contains business logic
- **Responsibilities**:
 - Business rules validation
 - Orchestrating multiple operations
 - Calling repositories and external services
 - Transaction management (`@Transactional`)
- **Example**: `CampaignDraftService.java` (interface) and `CampaignDraftServiceImpl.java` (implementation)


**3. Repository Layer** (MongoDB - No explicit repository classes)
- **Purpose**: Data access
- **How it works**: Uses `MongoTemplate` directly (Spring Data MongoDB)
- **Responsibilities**:
 - CRUD operations
 - Query building
 - Data persistence


**4. Model/DTO Layer** (`*DTO.java`)
- **Purpose**: Data structures for API communication
- **Responsibilities**:
 - Define request/response structure
 - Validation annotations (`@NotNull`, `@NotEmpty`, etc.)
 - JSON serialization/deserialization


**5. Entity Layer** (MongoDB Documents)
- **Purpose**: Database representation
- **Note**: In this project, DTOs are used directly as MongoDB documents (no separate entity layer)


---


## Campaign Creation Flow - Deep Dive


### Overview: Multi-Step Campaign Creation


Campaign creation happens in **multiple steps** (like a wizard):
1. **Step 1**: Basic Details (name, coupon code, tenant)
2. **Step 2**: Client Version Support (for online campaigns)
3. **Step 3**: Time Frame (start/end dates)
4. **Step 4**: Rules (conditions and actions)
5. **Step 5**: Rules Probability (if label type is "Probability")
6. **Step 6**: Budgets (for online campaigns)
7. **Step 7**: Terms and Conditions
8. **Final**: Submit (publishes campaign)


Each step saves to `campaign_draft` collection. Only after "Submit" does it move to `campaign` collection.


---


### 1. API Entry Point - Campaign Creation (Step 1)


#### Controller: `CampaignDraftController.java`


**File Location**: `promo-management/src/main/java/com/jubl/food/promo/management/campaigns/drafts/controllers/CampaignDraftController.java`


**Endpoint Details**:
```java
@PostMapping("/tenants/{tenantKey}/campaigns/drafts")
public ResponseEntity<String> campaignDraftSave(
   final @PathVariable(value = "tenantKey") String tenantKey,
   final @RequestBody @Validated({CampaignView.Campaign.class})
   @JsonView(CampaignView.Campaign.class) CampaignDTO campaign
)
```


**What This Means**:
- **HTTP Method**: `POST`
- **URL**: `/api/promo-management/ve1/tenants/{tenantKey}/campaigns/drafts`
- **Example**: `POST http://localhost:8022/api/promo-management/ve1/tenants/DOMINOS/campaigns/drafts`
- **Request Body**: JSON matching `CampaignDTO` structure
- **Response**: `{"uid": "generated-uuid"}` with HTTP 201 (Created)


**Annotations Explained**:


1. **`@RestController`** (class level)
  - Tells Spring this class handles HTTP requests
  - Combines `@Controller` + `@ResponseBody`


2. **`@RequestMapping("/ve1/")`** (class level)
  - Base path for all endpoints in this controller
  - `/ve1/` is the API version prefix


3. **`@PostMapping("/tenants/{tenantKey}/campaigns/drafts")`**
  - Maps HTTP POST requests to this method
  - `{tenantKey}` is a path variable (e.g., "DOMINOS")


4. **`@PathVariable("tenantKey")`**
  - Extracts `tenantKey` from URL path
  - Example: URL `/tenants/DOMINOS/...` → `tenantKey = "DOMINOS"`


5. **`@RequestBody`**
  - Tells Spring to deserialize JSON request body into `CampaignDTO` object
  - Spring uses Jackson library automatically


6. **`@Validated({CampaignView.Campaign.class})`**
  - Triggers validation based on `CampaignView.Campaign` group
  - Only validates fields annotated with `groups = CampaignView.Campaign.class`


7. **`@JsonView(CampaignView.Campaign.class)`**
  - Only includes fields marked with `@JsonView(CampaignView.Campaign.class)` in JSON
  - Filters out unnecessary fields


8. **`@Authorize(resource = "promoCampaign", action = "create")`**
  - Custom authorization annotation
  - Checks if user has permission to create campaigns
  - Implemented by security framework


**Controller Method Flow** (Lines 84-88):
```java
public ResponseEntity<String> campaignDraftSave(...) {
   ValidationUtility.compareTenantKeys(tenantKey, campaign.getTenantKey());
   return new ResponseEntity<>(
       "{\"uid\": \"" + campaignDraftService.saveCampaignDraft(tenantKey, campaign) + "\"}",
       HttpStatus.CREATED
   );
}
```


**What Happens**:
1. Validates tenant key matches between URL and request body
2. Calls service layer method `saveCampaignDraft()`
3. Returns UUID in JSON format with HTTP 201 status


---


### 2. Service Layer Journey


#### Service Interface: `CampaignDraftService.java`


**File Location**: `promo-management/src/main/java/com/jubl/food/promo/management/campaigns/drafts/CampaignDraftService.java`


**Method Signature** (Lines 105-112):
```java
@Transactional(transactionManager = "mongoTransactionManager")
default String saveCampaignDraft(final String tenantKey, final CampaignDTO campaign) {
   // create new campaign
   campaign.setUid(UUID.randomUUID().toString());
   campaign.setActive(campaign.getUid());
   getMongoTemplate().save(campaign, getCollectionName(tenantKey, CollectionType.CAMPAIGN_DRAFT));
   return campaign.getUid();
}
```


**Key Concepts**:


1. **`@Transactional`**
  - **What it does**: Ensures all database operations succeed or fail together (atomicity)
  - **Why needed**: If saving fails halfway, changes are rolled back
  - **`transactionManager = "mongoTransactionManager"`**: Uses MongoDB transaction manager (not JPA)


2. **`default` keyword**
  - Java 8+ feature: Interface can have default implementations
  - This method has a default implementation, but can be overridden


3. **`UUID.randomUUID().toString()`**
  - Generates unique identifier: `"a1b2c3d4-e5f6-7890-abcd-ef1234567890"`
  - Used as campaign `uid` (unique identifier)


4. **`campaign.setActive(campaign.getUid())`**
  - Sets `active` field to the campaign's own UID
  - **Why?** When `active == uid`, campaign is inactive (draft)
  - When `active == "true"`, campaign is active (published and enabled)


5. **`getMongoTemplate().save(...)`**
  - **MongoTemplate**: Spring's MongoDB access class (like JDBC for MongoDB)
  - **`save()`**: Inserts if new, updates if exists (upsert operation)
  - **Collection Name**: `getCollectionName(tenantKey, CollectionType.CAMPAIGN_DRAFT)`
    - Example: `"DOMINOS_campaign_draft"` (tenant-prefixed collection)


**Collection Naming Pattern**:
```java
// From AssortedUtility.getCollectionName()
// Format: "{tenantKey}_{collectionName}"
// Example: "DOMINOS_campaign_draft"
```


**Service Implementation**: `CampaignDraftServiceImpl.java`


**File Location**: `promo-management/src/main/java/com/jubl/food/promo/management/campaigns/drafts/CampaignDraftServiceImpl.java`


**Key Dependencies Injected** (Lines 81-115):
```java
@Autowired
private MongoTemplate mongoTemplate;        // MongoDB operations


@Autowired
private Validator validator;                 // Bean validation


@Autowired
private VersionConfig versionConfig;         // Version management


@Autowired
private MessageByLocaleService messageByLocaleService;  // Error messages


@Autowired
private KafkaTemplate<String, FileEvent> fileEventKafkaTemplate;  // Kafka producer


@Autowired
private CampaignSummarizer campaignSummarizer;  // Campaign summary updates


@Autowired
private AdminClient adminClient;            // External service client
```


**Dependency Injection Explained**:
- **`@Autowired`**: Spring automatically finds and injects these dependencies
- **How it works**: Spring scans for `@Component`, `@Service`, `@Repository` classes
- **Benefits**: Loose coupling, easier testing (can mock dependencies)


---


### 3. Data Layer Understanding


#### Database: MongoDB (NoSQL Document Database)


**Why MongoDB?**
- Campaigns are complex nested JSON structures
- No fixed schema - flexible for evolving requirements
- Better performance for document-based queries


**Collections Used**:


1. **`{tenantKey}_campaign_draft`**
  - **Purpose**: Stores campaigns being created/edited (work in progress)
  - **Example Document**:
    ```json
    {
      "_id": ObjectId("..."),
      "uid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "tenantKey": "DOMINOS",
      "couponCode": "SAVE50",
      "active": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",  // Inactive (draft)
      "details": {
        "labelType": "DISCOUNT",
        "rules": [...],
        ...
      },
      "createdAt": ISODate("2026-02-21T10:00:00Z"),
      "createdBy": "user@example.com"
    }
    ```


2. **`{tenantKey}_campaign`**
  - **Purpose**: Stores published/active campaigns
  - **Structure**: Same as draft, but `active` can be `"true"` or UID


3. **`{tenantKey}_campaign_history`**
  - **Purpose**: Version history (every update creates a new version)
  - **Additional Field**: `versionId` (integer, increments)


4. **`{tenantKey}_campaign_summary`**
  - **Purpose**: Denormalized summary for fast queries
  - **Fields**: `uid`, `couponCode`, `active`, `versionId`, etc.


**MongoDB Operations**:


```java
// Save (insert or update)
mongoTemplate.save(campaign, collectionName);


// Find by UID
Query query = new Query();
query.addCriteria(Criteria.where("uid").is(uid));
CampaignDTO campaign = mongoTemplate.findOne(query, CampaignDTO.class, collectionName);


// Delete
mongoTemplate.remove(query, collectionName);


// Count
long count = mongoTemplate.count(query, collectionName);
```


**Entity/Model Class**: `CampaignDTO.java`


**File Location**: `promo-management/src/main/java/com/jubl/food/promo/management/common/models/campaigns/CampaignDTO.java`


**Key Fields** (Lines 49-126):
```java
@MongoId
private ObjectId id;              // MongoDB's internal ID


private String uid;               // Business identifier (UUID)


@NotEmpty(groups = CampaignView.Campaign.class)
private String tenantKey;         // Tenant identifier


@NotEmpty(groups = CampaignView.Campaign.class)
@Pattern(regexp = "^[^ a-z]+$")  // No lowercase letters
private String couponCode;        // User-facing code (e.g., "SAVE50")


private String startDate;          // ISO date string
private String endDate;            // ISO date string


private String active;            // "true" or UID (inactive)


@NotNull
@Valid
private CampaignDetailDTO details; // Nested details object


private List<ClientVersionSupport> clientVersionSupport;  // App version support


private CampaignStoresInfo storesInfo;  // Store selection


// ... many more fields
```


**Annotations Explained**:


1. **`@MongoId`**
  - Marks field as MongoDB document ID
  - MongoDB uses `_id` field internally


2. **`@NotEmpty`**
  - Validation: Field cannot be null or empty
  - `groups = CampaignView.Campaign.class`: Only validates in Campaign view group


3. **`@Pattern(regexp = "^[^ a-z]+$")`**
  - Validation: Coupon code cannot contain lowercase letters
  - Example: "SAVE50" ✅, "save50" ❌


4. **`@Valid`**
  - Cascades validation to nested object (`CampaignDetailDTO`)


5. **`@JsonView(CampaignView.Campaign.class)`**
  - Controls which fields appear in JSON response
  - Different views for different API endpoints


**Inheritance**: `CampaignDTO extends Auditable<String>`


**Auditable Fields** (from parent class):
```java
private String createdBy;         // User who created
private Date createdAt;           // Creation timestamp
private String lastModifiedBy;   // User who last modified
private Date lastModifiedAt;      // Last modification timestamp
```


**No Traditional Repository Interface**


Unlike JPA (MySQL), this project uses `MongoTemplate` directly:
- **Why?** More flexibility for complex queries
- **Alternative**: Could use `MongoRepository` interface (Spring Data MongoDB)


---


### 4. External Dependencies During Campaign Creation


#### A. Admin Service Call (Tenant Validation)


**When**: During campaign submission (not creation)


**Client**: `AdminClient.java`


**Method**: `fetchTenantByTenantKey(tenantKey)`


**Purpose**:
- Validates tenant exists
- Fetches tenant configurations (thresholds, timezone, etc.)


**How It Works**:
```java
// In CampaignDraftServiceImpl.submitCampaignDraft()
TenantDetailResponseDTO tenantDetails = adminClient.fetchTenantByTenantKey(tenantKey);


// Uses tenant configuration for validation
String thresholdLimit = tenantDetails.getTenantConfigurations()
   .get(Constants.THRESHOLD_ACTIVE_OFFER);
```


**HTTP Call**:
```
GET http://localhost:8083/nxg-admin/vi1/tenants/{tenantKey}
```


#### B. Kafka Event Publishing


**When**: During campaign submission (for personalized campaigns)


**Topics Used**:


1. **`personalized-coupon-file-event`**
  - **When**: Campaign has file upload for personalized coupons
  - **Purpose**: Triggers file processing service
  - **Code Location**: `CampaignDraftServiceImpl.publishPersonalCampaignFileEvents()` (Lines 505-562)


2. **`open-coupon-topic`** (ShareableCouponEvent)
  - **When**: Campaign generates shareable/open coupons
  - **Purpose**: Triggers coupon generation service
  - **Batch Processing**: Sends events in batches (1000 coupons per event)


**Example Kafka Event**:
```java
FileEvent filesEvent = new FileEvent();
filesEvent.setCampaignId(pUID);
filesEvent.setTenantKey(tenantKey);
filesEvent.setCouponCode(campaignDraft.getCouponCode());
filesEvent.setVersionId(version);
fileEventKafkaTemplate.send(kafkaTopicConfig.getFileEventTopic(), filesEvent);
```


**Why Async?**
- File processing can take minutes/hours
- Don't block API response
- Better scalability


#### C. Value Group Integration


**When**: During rules update (Step 4) and submission


**Service**: `CampaignValueGroupIntegrator`


**Purpose**:
- Validates value group keys exist
- Updates campaign rules with actual values from value groups
- Maintains links between campaigns and value groups


**Flow**:
```java
// Extract value group UIDs from campaign rules
Set<String> valueGroupUIDs = campaignValueGroupIntegrator
   .extractValueGroupUIDs(campaignDetail);


// Fetch value groups
List<ValueGroups> valueGroups = campaignValueGroupIntegrator
   .getValueGroups(tenantKey, valueGroupUIDs);


// Validate and update campaign
campaignValueGroupIntegrator.validateValueGroupKeysExist(valueGroupUIDs, valueGroups);
campaignValueGroupIntegrator.updateCampaignValues(campaign.getDetails().getRules(), valueGroup);
```


---


### 5. Complete Flow with Code Navigation


#### Step-by-Step: Campaign Creation (Step 1)


**Request Flow**:


```
1. HTTP Request
  ↓
  POST /api/promo-management/ve1/tenants/DOMINOS/campaigns/drafts
  Body: { "tenantKey": "DOMINOS", "couponCode": "SAVE50", "details": {...} }
  ↓
2. CampaignDraftController.campaignDraftSave()
  - Validates tenant key match
  - Calls service
  ↓
3. CampaignDraftService.saveCampaignDraft()
  - Generates UUID
  - Sets active = uid (inactive)
  - Saves to MongoDB
  ↓
4. MongoDB Save Operation
  - Collection: "DOMINOS_campaign_draft"
  - Document inserted
  ↓
5. Response
  - Returns: {"uid": "a1b2c3d4-..."}
  - HTTP 201 Created
```


**Code Trace**:


1. **Controller** (`CampaignDraftController.java:84-88`)
  ```java
  @PostMapping("/tenants/{tenantKey}/campaigns/drafts")
  public ResponseEntity<String> campaignDraftSave(...) {
      ValidationUtility.compareTenantKeys(tenantKey, campaign.getTenantKey());
      return new ResponseEntity<>("{\"uid\": \"" +
          campaignDraftService.saveCampaignDraft(tenantKey, campaign) + "\"}",
          HttpStatus.CREATED);
  }
  ```


2. **Service** (`CampaignDraftService.java:105-112`)
  ```java
  @Transactional(transactionManager = "mongoTransactionManager")
  default String saveCampaignDraft(final String tenantKey, final CampaignDTO campaign) {
      campaign.setUid(UUID.randomUUID().toString());
      campaign.setActive(campaign.getUid());
      getMongoTemplate().save(campaign,
         getCollectionName(tenantKey, CollectionType.CAMPAIGN_DRAFT));
      return campaign.getUid();
  }
  ```


3. **MongoDB Operation** (Spring Data MongoDB)
  - `MongoTemplate.save()` converts `CampaignDTO` to BSON document
  - Inserts into collection: `"DOMINOS_campaign_draft"`


**DTO Transformations**:


- **Request JSON → CampaignDTO**: Jackson deserializes JSON
- **CampaignDTO → MongoDB Document**: Spring Data MongoDB serializes
- **MongoDB Document → Response JSON**: Jackson serializes


**No Explicit Mapping Needed**: DTOs are used directly as MongoDB documents (no separate entity layer).


---


#### Complete Campaign Submission Flow


**When**: User clicks "Submit" after completing all steps


**Endpoint**: `POST /tenants/{tenantKey}/campaigns/drafts/{uid}/submit`


**Controller** (`CampaignDraftController.java:224-227`):
```java
@PostMapping("/tenants/{tenantKey}/campaigns/drafts/{uid}/submit")
public void submitCampaignDraft(final String tenantKey, final String uid) {
   campaignDraftService.submitCampaignDraft(tenantKey, uid);
}
```


**Service Implementation** (`CampaignDraftServiceImpl.java:423-496`):


**Step-by-Step Breakdown**:


```java
@Transactional(transactionManager = "mongoTransactionManager")
public void submitCampaignDraft(final String tenantKey, final String pUID) {
  
   // 1. Fetch campaign draft
   CampaignVersionDTO campaignDraft = findCampaignByUID(...)
       .orElseThrow(() -> new PromoException(...));
  
   // 2. Validate all fields
   Set<ConstraintViolation<CampaignVersionDTO>> errors =
       validator.validate(campaignDraft, CampaignView.Submit.class);
   if (!errors.isEmpty()) {
       throw new ConstraintViolationException(errors);
   }
  
   // 3. Additional validations
   validateRandomRule(...);  // If probability campaign
   validateChannelVSChannelGroup(...);
  
   // 4. Get next version number
   int version = getVersionConfig().getNextVersion(...);
   campaignDraft.setVersionId(version);
  
   // 5. Publish Kafka events (if personalized campaign)
   publishPersonalCampaignFileEvents(tenantKey, pUID, campaignDraft, ...);
  
   // 6. Update value groups in campaign
   campaignValueGroupIntegrator.updateCampaignValues(...);
  
   // 7. Save to campaign collection (published)
   getMongoTemplate().save(campaignDraft,
       getCollectionName(tenantKey, CollectionType.CAMPAIGN));
  
   // 8. Insert into history (for versioning)
   campaignDraft.setId(null);  // Clear ID for new insert
   getMongoTemplate().insert(campaignDraft,
       getCollectionName(tenantKey, CollectionType.CAMPAIGN_HISTORY));
  
   // 9. Delete from draft
   getMongoTemplate().remove(query,
       getCollectionName(tenantKey, CollectionType.CAMPAIGN_DRAFT));
  
   // 10. Update campaign summary (denormalized data)
   getCampaignSummarizer().upsertCampaignSummary(tenantKey, campaignDraft);
  
   // 11. Update value group links
   campaignValueGroupIntegrator.updateCampaignValueGroupLinks(...);
}
```


**Why This Order?**
1. Validate first (fail fast)
2. Generate version before saving
3. Publish events before deleting draft (in case rollback needed)
4. Save to campaign (published state)
5. Save history (versioning)
6. Delete draft (cleanup)
7. Update summary (for fast queries)


**Transaction Management**:
- All operations wrapped in `@Transactional`
- If any step fails, entire operation rolls back
- MongoDB transactions ensure atomicity


---


## Campaign Update Flow


### Overview


Campaign updates follow a **versioning pattern**:
- Each update creates a new **version**
- Previous versions stored in `campaign_history`
- Current version in `campaign` collection
- Can restore previous versions


### Update Endpoints


#### 1. Update Campaign Details (Step 1)


**Endpoint**: `PUT /tenants/{tenantKey}/campaigns/drafts/{uid}/details`


**Controller** (`CampaignDraftController.java:97-105`):
```java
@PutMapping("/tenants/{tenantKey}/campaigns/drafts/{uid}/details")
public void updateCampaignDraftDetails(
   final @PathVariable("tenantKey") String tenantKey,
   final @PathVariable("uid") String uid,
   final @RequestBody @Validated({CampaignView.Campaign.class})
   @JsonView(CampaignView.Campaign.class) CampaignDTO campaign
) {
   campaign.setUid(uid);
   campaignDraftService.updateCampaignDetailsDraft(tenantKey, campaign);
}
```


**Service Implementation** (`CampaignDraftService.java:120-168`):


**Key Logic**:
```java
@Transactional(transactionManager = "mongoTransactionManager")
default void updateCampaignDetailsDraft(final String tenantKey, final CampaignDTO pCampaign) {
   // 1. Find existing draft
   Optional<CampaignDTO> campaignDraft = findCampaignByUID(
       pCampaign.getUid(),
       getCollectionName(tenantKey, CollectionType.CAMPAIGN_DRAFT),
       CampaignDTO.class
   );
  
   if (campaignDraft.isPresent()) {
       // 2. Check if published version exists
       Optional<CampaignVersionDTO> campaignVersionDTO = findCampaignByUID(
           pCampaign.getUid(),
           getCollectionName(tenantKey, CollectionType.CAMPAIGN),
           CampaignVersionDTO.class
       );
      
       // 3. Validate personalizable field hasn't changed (if published)
       campaignVersionDTO.ifPresent(versionDTO ->
           ValidationUtility.validateCampaignPersonalizable(
               versionDTO.getDetails(), pCampaign.getDetails()
           )
       );
      
       // 4. Validate savings messages
       ValidationUtility.validateSavingsMessage(...);
      
       // 5. Preserve existing rules (if any)
       if (!CollectionUtils.isEmpty(campaignDraft.get().getDetails().getRules())) {
           pCampaign.getDetails().setRules(campaignDraft.get().getDetails().getRules());
       }
      
       // 6. Update draft
       campaignDraft.get().setDetails(pCampaign.getDetails());
       if (StringUtils.isNotEmpty(pCampaign.getCouponCode())) {
           campaignDraft.get().setCouponCode(pCampaign.getCouponCode());
       }
      
       // 7. Save updated draft
       getMongoTemplate().save(campaignDraft.get(),
           getCollectionName(tenantKey, CollectionType.CAMPAIGN_DRAFT));
          
   } else {
       // 8. If no draft exists, fetch from published campaign
       CampaignDTO campaign = findCampaignByUID(...)
           .orElseThrow(...);
      
       // 9. Copy published campaign to draft for editing
       // ... update fields ...
      
       // 10. Save as draft
       getMongoTemplate().save(campaign,
           getCollectionName(tenantKey, CollectionType.CAMPAIGN_DRAFT));
   }
}
```


**Key Differences from Creation**:


1. **Finds Existing Draft**: Uses `findCampaignByUID()` to get current draft
2. **Preserves Rules**: Doesn't overwrite rules if they exist (rules updated separately)
3. **Validates Published Version**: If campaign is published, validates constraints
4. **Creates Draft from Published**: If updating published campaign, creates draft first


#### 2. Update Published Campaign (Toggle Active Status)


**Endpoint**: `PUT /tenants/{tenantKey}/campaigns/{uid}/toggle/{status}`


**Controller** (`CampaignController.java:102-108`):
```java
@PutMapping("/tenants/{tenantKey}/campaigns/{uid}/toggle/{status}")
public void updateCampaignStatus(
   final @PathVariable("tenantKey") String tenantKey,
   final @PathVariable("uid") String uid,
   final @PathVariable("status") boolean status
) {
   campaignService.updateCampaignActiveStatus(tenantKey, uid, status);
}
```


**Service Implementation** (`CampaignServiceImpl.java:202-247`):


```java
@Transactional(transactionManager = "mongoTransactionManager")
public void updateCampaignActiveStatus(final String key, final String uid, final boolean status) {
   // 1. Find campaign
   CampaignVersionDTO campaign = findCampaignByUID(uid,
       getCollectionName(key, CollectionType.CAMPAIGN), CampaignVersionDTO.class)
       .orElseThrow(...);
  
   // 2. Validate modification access (if approval required)
   if (status && !StringUtils.equals("skip", getApprovalAccessCheck())) {
       ValidationUtility.validateModificationAccess(campaign.getLastModifiedBy());
   }
  
   // 3. Check tenant thresholds (if activating)
   if (status) {
       TenantDetailResponseDTO tenantDetails = getTenantDetailsByTenantKey(key);
      
       // Check visible campaign threshold
       if (campaign.getDetails().isVisible()) {
           String thresholdLimit = tenantDetails.getTenantConfigurations()
               .get(Constants.THRESHOLD_DISPLAY_OFFER);
           if (!StringUtils.isEmpty(thresholdLimit)) {
               CampaignFilter filter = new CampaignFilter();
               filter.setActive(true);
               filter.setVisible(true);
               Long count = getCampaignsCount(key, filter);
               if (count > Long.parseLong(thresholdLimit)) {
                   throw new PromoException(...);  // Too many active campaigns
               }
           }
       }
      
       // Check total active campaign threshold
       String thresholdLimit = tenantDetails.getTenantConfigurations()
           .get(Constants.THRESHOLD_ACTIVE_OFFER);
       // ... similar validation ...
      
       campaign.setActive(Boolean.TRUE.toString());  // Activate
   } else {
       campaign.setActive(uid);  // Deactivate (set to own UID)
   }
  
   // 4. Save campaign
   try {
       getMongoTemplate().save(campaign, getCollectionName(key, CollectionType.CAMPAIGN));
      
       // 5. Save toggle audit (history)
       getMongoTemplate().save(
           getCampaignToggleAuditSaver().generateCampaignToggleAuditData(campaign),
           getCollectionName(key, CollectionType.CAMPAIGN_TOGGLE_HISTORY)
       );
      
       // 6. Update summary
       getCampaignSummarizer().toggleCampaignSummary(campaign);
      
   } catch (DuplicateKeyException e) {
       throw new PromoException(HttpStatus.CONFLICT,
           "Duplicate coupon code: " + campaign.getCouponCode());
   }
}
```


**What Happens to Existing Data**:


1. **No Version Created**: Toggle doesn't create new version (just updates `active` field)
2. **Audit Trail**: Saves toggle history to `campaign_toggle_history` collection
3. **Summary Updated**: Updates denormalized summary for fast queries
4. **Duplicate Check**: MongoDB unique index on `couponCode` prevents duplicates


### Versioning and Optimistic Locking


**Version Management**:


- **Version ID**: Integer that increments with each update
- **Version Config Collection**: `{tenantKey}_version_config`
 - Stores: `{ "uid": "campaign-uid", "versionId": 5 }`
 - Used to generate next version number


**How Versioning Works**:


```java
// Get next version
int version = getVersionConfig().getNextVersion(
   getCollectionName(tenantKey, CollectionType.CAMPAIGN_HISTORY),
   campaignDraft.getUid(),
   getCollectionName(tenantKey, CollectionType.VERSION_CONFIG)
);


// Set version on campaign
campaignDraft.setVersionId(version);


// Save to campaign (current version)
mongoTemplate.save(campaignDraft, campaignCollection);


// Save to history (all versions)
campaignDraft.setId(null);  // Clear MongoDB ID
mongoTemplate.insert(campaignDraft, historyCollection);
```


**Restore Previous Version**:


**Endpoint**: `POST /tenants/{tenantKey}/campaigns/{uid}/versions/{versionId}/restore`


**Flow**:
1. Fetch campaign history by `versionId`
2. Validate version can be restored
3. Generate new version number
4. Move current campaign to history
5. Restore old version as current


**No Optimistic Locking**:
- This project doesn't use optimistic locking (no `@Version` field)
- Relies on MongoDB transactions for consistency
- Could be added for concurrent update prevention


---


## Additional Information


### 1. Common Configuration Files


#### `application.yml`


**Location**: `promo-management/src/main/resources/application.yml`


**Key Sections**:


1. **Server Configuration** (Lines 1-12)
  ```yaml
  server:
    port: 8022
    servlet:
      context-path: /api/promo-management
  ```
  - Port: 8022
  - Base path: `/api/promo-management`


2. **Database Connections** (Lines 130-153)
  - MySQL: JDBC URL, username, password
  - MongoDB: Connection URI
  - Redis: Host and port


3. **Kafka Configuration** (Lines 62-124)
  - Bootstrap servers
  - Topic names
  - Producer/Consumer settings
  - Retry configuration


4. **External Service URLs** (Lines 44-49)
  ```yaml
  service:
    baseURL:
      promoCalculator: "http://localhost:8085/"
      promoAdmin: "http://localhost:8083/nxg-admin/"
  ```


5. **Caching** (Lines 125-192)
  - Redis cache settings
  - Caffeine cache configurations


6. **Feature Flags** (Lines 303-337)
  ```yaml
  entitlement:
    auth:
      config:
        enable: false  # Feature flag
  ```


#### `bootstrap.yml`


**Location**: `promo-management/src/main/resources/bootstrap.yml`


**Purpose**:
- Loaded before `application.yml`
- Contains Spring Cloud Config settings
- Application name: `nextgen-promo-mgmt`


### 2. Logging and Error Handling Patterns


#### Logging


**Framework**: SLF4J with Logback


**Configuration**: `logback-spring.xml`


**Usage in Code**:
```java
@Slf4j  // Lombok annotation - creates 'log' field
public class CampaignDraftServiceImpl {
  
   public void someMethod() {
       log.debug("Debug message: {}", variable);
       log.info("Info message: {}", variable);
       log.error("Error message", exception);
   }
}
```


**Log Levels** (from `application.yml`):
```yaml
logging:
 level:
   web: DEBUG
   org.hibernate.SQL: INFO
   org.springframework.data.mongodb.core.MongoTemplate: DEBUG
```


#### Error Handling


**Custom Exception**: `PromoException`


**Location**: Inherited from `promo-utils` library


**Usage**:
```java
throw new PromoException(
   HttpStatus.BAD_REQUEST,
   messageByLocaleService.getMessage("err.campaign.id.invalid", uid)
);
```


**Error Response Format**:
```json
{
 "status": 400,
 "message": "Campaign ID is invalid: abc123",
 "timestamp": "2026-02-21T10:00:00Z"
}
```


**Global Exception Handler**:
- Likely in `promo-utils` library
- Handles `PromoException` and converts to JSON response


### 3. Design Patterns Used


#### 1. **Service Layer Pattern**
- **Interface + Implementation**: `CampaignDraftService` (interface) + `CampaignDraftServiceImpl` (implementation)
- **Why**: Allows multiple implementations, easier testing


#### 2. **DTO Pattern**
- **Purpose**: Separate API contracts from database structure
- **In this project**: DTOs used directly as MongoDB documents (no separate entity)


#### 3. **Repository Pattern** (Implicit)
- **MongoTemplate**: Acts as repository
- **Alternative**: Could use `MongoRepository` interface


#### 4. **Dependency Injection**
- **Spring's `@Autowired`**: Injects dependencies automatically
- **Benefits**: Loose coupling, testability


#### 5. **Transaction Management**
- **`@Transactional`**: Declarative transactions
- **MongoDB Transactions**: Ensures atomicity


#### 6. **Builder Pattern** (via Lombok)
- **`@Builder`**: Generates builder methods
- **Usage**: `CampaignDTO.builder().uid("...").build()`


#### 7. **Strategy Pattern** (Implicit)
- **Different campaign types**: Online vs Offline
- **Different services**: `CampaignDraftService` vs `OfflineCampaignDraftService`


### 4. Common Pitfalls for Beginners


#### 1. **Forgetting `@Transactional`**
- **Problem**: Operations not atomic, partial updates possible
- **Solution**: Always use `@Transactional` for multi-step operations


#### 2. **Mixing Tenant Keys**
- **Problem**: Saving to wrong tenant's collection
- **Solution**: Always validate `tenantKey` matches between URL and request body


#### 3. **Not Handling Optional Values**
- **Problem**: `NullPointerException` when optional is empty
- **Solution**: Use `Optional.isPresent()` or `orElseThrow()`


#### 4. **Ignoring Validation Groups**
- **Problem**: Wrong validations triggered
- **Solution**: Match `@Validated` groups with `@JsonView` groups


#### 5. **MongoDB ID Confusion**
- **Problem**: Trying to set `_id` when inserting (should be null)
- **Solution**: Clear `id` before insert: `campaign.setId(null)`


#### 6. **Kafka Event Ordering**
- **Problem**: Events processed out of order
- **Solution**: Use partition keys, ensure transactional publishing


#### 7. **Collection Name Mismatch**
- **Problem**: Querying wrong collection
- **Solution**: Always use `getCollectionName(tenantKey, CollectionType.XXX)`


#### 8. **Not Clearing Draft After Submit**
- **Problem**: Draft remains, causing confusion
- **Solution**: Always delete draft after successful submission


### 5. Testing Tips


#### Unit Tests
- **Mock Dependencies**: Use `@MockBean` for `MongoTemplate`, `AdminClient`, etc.
- **Test Service Logic**: Focus on business logic, not database operations


#### Integration Tests
- **Embedded MongoDB**: Use `@DataMongoTest` or embedded MongoDB
- **Test Transactions**: Verify rollback on errors


#### API Tests
- **MockMvc**: Test controllers without starting full server
- **Test Validation**: Send invalid data, verify error responses


### 6. Key Spring Boot Concepts Explained


#### Dependency Injection
- **What**: Spring automatically provides dependencies
- **How**: Annotate with `@Autowired` or constructor injection
- **Example**:
 ```java
 @Autowired
 private MongoTemplate mongoTemplate;  // Spring finds and injects
 ```


#### Bean Scopes
- **Singleton** (default): One instance per application
- **Request**: New instance per HTTP request
- **Prototype**: New instance every time


#### AOP (Aspect-Oriented Programming)
- **`@Transactional`**: Uses AOP to wrap methods in transactions
- **`@Retryable`**: Retries method on failure


#### Spring Data MongoDB
- **MongoTemplate**: Low-level MongoDB operations
- **MongoRepository**: Higher-level abstraction (not used here)
- **Query DSL**: Type-safe queries (used via `Query` and `Criteria`)


---


## Summary


### Campaign Creation Flow (Quick Reference)


1. **POST** `/tenants/{tenantKey}/campaigns/drafts` → Creates draft
2. **PUT** `/tenants/{tenantKey}/campaigns/drafts/{uid}/details` → Updates details
3. **PUT** `/tenants/{tenantKey}/campaigns/drafts/{uid}/time` → Sets timeframe
4. **PUT** `/tenants/{tenantKey}/campaigns/drafts/{uid}/rules` → Sets rules
5. **PUT** `/tenants/{tenantKey}/campaigns/drafts/{uid}/budgets` → Sets budgets
6. **PUT** `/tenants/{tenantKey}/campaigns/drafts/{uid}/tncs` → Sets T&Cs
7. **POST** `/tenants/{tenantKey}/campaigns/drafts/{uid}/submit` → Publishes campaign


### Campaign Update Flow (Quick Reference)


1. **PUT** `/tenants/{tenantKey}/campaigns/drafts/{uid}/details` → Updates draft
2. **POST** `/tenants/{tenantKey}/campaigns/drafts/{uid}/submit` → Creates new version
3. **PUT** `/tenants/{tenantKey}/campaigns/{uid}/toggle/{status}` → Activates/deactivates


### Key Collections


- `{tenantKey}_campaign_draft` - Work in progress
- `{tenantKey}_campaign` - Published campaigns
- `{tenantKey}_campaign_history` - Version history
- `{tenantKey}_campaign_summary` - Denormalized summary


### Key Concepts


- **UUID**: Business identifier (`uid`)
- **Version ID**: Integer version number
- **Active Field**: `"true"` = active, UID = inactive
- **Tenant Isolation**: Each tenant has separate collections
- **Transactions**: MongoDB transactions ensure atomicity
- **Async Processing**: Kafka events for long-running tasks


---


## Next Steps


1. **Run the Application**: Follow setup instructions above
2. **Explore Swagger UI**: `http://localhost:8022/api/promo-management/swagger-ui.html`
3. **Create a Test Campaign**: Use Postman/curl to create a campaign
4. **Read the Code**: Start with `CampaignDraftController`, trace through service
5. **Set Breakpoints**: Debug through the flow to understand execution
6. **Check Logs**: Enable DEBUG logging to see MongoDB queries


---


**Questions?** Refer to:
- Spring Boot Documentation: https://spring.io/projects/spring-boot
- MongoDB Java Driver: https://mongodb.github.io/mongo-java-driver/
- Spring Data MongoDB: https://spring.io/projects/spring-data-mongodb


---


*End of Knowledge Transfer Document*



