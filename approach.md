# Reverse-Engineering Methodology: Universal Workflow Mapping

## Executive Summary

This document outlines a systematic, platform-agnostic methodology for mapping user intents to API workflows in any SaaS platform. The approach combines automated discovery techniques with semantic analysis to create machine-readable workflow representations.

---

## 1. Core Methodology: The Four-Layer Analysis Framework

### Layer 1: Intent Decomposition
**Goal:** Break down user request into atomic operations

**Process:**
1. **Entity Identification:** Extract nouns (e.g., "shipping zone", "countries", "shipping method")
2. **Action Identification:** Extract verbs (e.g., "create", "add", "configure")
3. **Constraint Extraction:** Identify conditions (e.g., "only applies to this zone", "$15")
4. **Dependency Mapping:** Determine operational order (e.g., zone must exist before adding countries)

**Example from Track B:**
```
User Intent: "Set up a new shipping zone for 'Oceania'. Add 'Australia' and 'New Zealand' 
             to this zone. Then, configure a 'Flat Rate' shipping method of $15 that only 
             applies to this zone."

Decomposition:
- Entity 1: Shipping Zone (name: "Oceania")
- Entity 2: Countries (Australia, New Zealand)
- Entity 3: Shipping Method (type: Flat Rate, amount: $15)
- Actions: CREATE zone → ADD countries → CREATE shipping method
- Dependencies: shipping_method.zone_id depends_on zone.id
```

### Layer 2: UI Instrumentation & Discovery
**Goal:** Map the visible user interface to underlying state changes

**Techniques Applied:**

#### 2.1 Browser DevTools Network Analysis
```
Method: Monitor XHR/Fetch requests during manual UI interaction
Tools: Chrome DevTools → Network Tab → Filter: XHR
Steps:
  1. Open DevTools before any interaction
  2. Perform UI action (e.g., click "Create Zone")
  3. Capture request payload, headers, response
  4. Document the GraphQL operation name and variables
```

**Discovery Pattern:**
```
UI Action: Click "Create Zone" button
↓
Network Event: POST /admin-api
↓
Request Payload: 
{
  "operationName": "CreateZone",
  "query": "mutation CreateZone($input: CreateZoneInput!) {...}",
  "variables": {"input": {"name": "Test"}}
}
↓
API Mapping: createZone mutation identified
```

#### 2.2 DOM Inspection for Form Binding
```
Method: Inspect form elements to understand data structure
Tools: Elements panel → Event Listeners
Process:
  1. Right-click form field → Inspect
  2. Check 'name' attribute (often matches API field)
  3. Review JavaScript event handlers
  4. Trace data binding to state management (React/Vue/Angular)
```

#### 2.3 GraphQL Schema Introspection
```graphql
query IntrospectionQuery {
  __schema {
    mutationType {
      name
      fields {
        name
        args {
          name
          type {
            name
            kind
          }
        }
      }
    }
  }
}
```

**Why This Matters:**
- Reveals ALL available mutations (not just documented ones)
- Shows required vs optional fields
- Identifies input types and their structures

### Layer 3: Semantic Gap Bridging
**Goal:** Map human language to API terminology

**Algorithm:**

```python
def map_semantic_gap(user_term, api_schema):
    """
    Multi-strategy semantic mapping algorithm
    """
    strategies = [
        exact_match,           # Direct string match
        fuzzy_match,           # Levenshtein distance < threshold
        synonym_expansion,     # NLP-based synonym matching
        contextual_inference,  # Use surrounding context
        documentation_search   # Grep through API docs
    ]
    
    for strategy in strategies:
        result = strategy(user_term, api_schema)
        if result.confidence > THRESHOLD:
            return result
    
    return manual_review_required(user_term)
```

**Real Examples from Vendure:**

| User Term | API Field | Discovery Method | Notes |
|-----------|-----------|------------------|-------|
| "$15" | `1500` (integer) | Documentation + Testing | Currency stored in cents |
| "Flat Rate" | `calculator.code: "flat-rate"` | Network tab inspection | Calculator is configurable operation |
| "Oceania" | `zone.name` (string) | Direct mapping | No transformation needed |
| "only applies to this zone" | `checker.code: "shipping-zone-checker"` | Schema introspection + docs | Checker determines eligibility |
| "Australia" | Country ID lookup | Query countries first | Names must be resolved to IDs |

**Challenge: The "Unlimited Stock" Problem**
```
User says: "Unlimited Stock"
Possible API mappings:
  - stockLevel: null
  - trackInventory: false
  - stockOnHand: -1
  - useGlobalStock: true

Solution: Test each hypothesis in sandbox, observe system behavior
Result: trackInventory: false (confirmed via testing)
```

### Layer 4: Dependency Graph Construction
**Goal:** Model the execution flow with data dependencies

**Graph Theory Approach:**
```
Nodes = Workflow Steps
Edges = Data Dependencies

Algorithm:
1. Create directed acyclic graph (DAG)
2. For each step S:
   - Add edge from S to S' if S.output is required by S'.input
3. Topological sort to determine execution order
4. Validate no cycles exist
```

**Example Dependency Chain:**
```
Step 1 (Query Countries)
  ↓ [outputs: australia_id, new_zealand_id]
  ↓
Step 2 (Create Zone)     Step 3 (Add Countries to Zone)
  ↓ [outputs: zone_id]  ←  ← [requires: zone_id, country_ids]
  ↓                         
  ↓                         
Step 4 (Create Shipping Method)
  ← [requires: zone_id]
```

**Implementation in JSON:**
```json
{
  "step_3": {
    "dependencies": {
      "data_flow": [
        {"from": "step_1.australia_country_id", "to": "step_3.inputs.memberIds[0]"},
        {"from": "step_2.zone_id", "to": "step_3.inputs.zoneId"}
      ]
    }
  }
}
```

---

## 2. Platform-Agnostic Generalization Strategy

### The Universal Workflow Mapping Algorithm

**Core Principle:** Every SaaS platform has the same fundamental architecture:
```
User Interface → State Management → API Layer → Database
```

**Generalized Discovery Process:**

#### Step 1: API Technology Detection
```python
def detect_api_technology(domain):
    """
    Identify the API paradigm used by the platform
    """
    detection_methods = {
        "graphql": lambda: check_for_endpoint("/graphql") or 
                          check_headers("content-type", "application/graphql"),
        "rest": lambda: analyze_url_patterns(["/api/v1/", "/api/v2/"]),
        "jsonrpc": lambda: check_request_body_structure({"jsonrpc": "2.0"}),
        "soap": lambda: check_headers("content-type", "text/xml")
    }
    
    return run_detection_methods(detection_methods)
```

**Platform Examples:**
- **Vendure:** GraphQL (detected via `/admin-api` endpoint)
- **Salesforce:** REST + SOAP (multiple versions)
- **Jira:** REST v3 + v2 (hybrid)
- **HubSpot:** REST (OAuth-based)

#### Step 2: Schema Discovery

**For GraphQL (Vendure, Shopify, GitHub):**
```graphql
# Introspection query reveals entire API surface
query { __schema { types { name fields { name type } } } }
```

**For REST (Salesforce, Jira, HubSpot):**
```python
def discover_rest_schema(base_url):
    """
    Multi-pronged REST API discovery
    """
    methods = [
        check_openapi_spec(f"{base_url}/swagger.json"),
        check_openapi_spec(f"{base_url}/openapi.json"),
        check_api_documentation(f"{base_url}/docs"),
        crawl_network_requests(),  # Fallback: observe real requests
    ]
    
    return aggregate_schema(methods)
```

**Example - Salesforce:**
```bash
# Describe all objects
GET /services/data/v58.0/sobjects/

# Describe specific object schema
GET /services/data/v58.0/sobjects/Lead/describe/
```

#### Step 3: UI-to-API Correlation

**Universal Pattern Matching:**

| UI Pattern | API Pattern | Detection Heuristic |
|------------|-------------|---------------------|
| "Create" button | POST /resource or createResource mutation | Button text + network POST |
| "Save" button | PUT /resource/:id or updateResource mutation | Button in edit form + PUT/PATCH |
| "Delete" button | DELETE /resource/:id or deleteResource mutation | Destructive action + DELETE |
| Dropdown/Select | GET /resource/options | Population before form render |
| Autocomplete | GET /resource?search=query | Debounced query on keypress |

**Implementation:**
```python
def correlate_ui_to_api(ui_action, network_logs):
    """
    Match UI interactions to API calls using temporal correlation
    """
    ui_timestamp = ui_action.timestamp
    
    # Find API calls within 500ms window
    candidate_calls = [
        call for call in network_logs 
        if abs(call.timestamp - ui_timestamp) < 500
    ]
    
    # Score by correlation strength
    scores = []
    for call in candidate_calls:
        score = calculate_correlation(
            ui_action.element_id,
            call.request_payload,
            ui_action.form_data
        )
        scores.append((call, score))
    
    return max(scores, key=lambda x: x[1])
```

#### Step 4: Field Name Similarity Matching

**When documentation is sparse or non-existent:**

```python
from difflib import SequenceMatcher

def find_matching_api_field(ui_label, api_fields):
    """
    Fuzzy match UI labels to API field names
    """
    transformations = [
        lambda x: x.lower().replace(" ", ""),        # "Product Name" → "productname"
        lambda x: x.lower().replace(" ", "_"),       # "Product Name" → "product_name"
        lambda x: camelCase(x),                      # "Product Name" → "productName"
        lambda x: snake_case(x),                     # "Product Name" → "product_name"
    ]
    
    best_match = None
    best_score = 0
    
    for field in api_fields:
        for transform in transformations:
            transformed_label = transform(ui_label)
            similarity = SequenceMatcher(None, transformed_label, field).ratio()
            
            if similarity > best_score:
                best_score = similarity
                best_match = field
    
    return best_match if best_score > 0.7 else None
```

**Example Application:**
```
UI Label: "Discount Percentage"
API Fields: ["discountPercent", "discount_rate", "percentOff", "rate"]

Results:
  discountPercent: 0.85 ✓ (best match)
  discount_rate: 0.65
  percentOff: 0.58
  rate: 0.20
```

---

## 3. Handling Platform-Specific Quirks

### Type System Discovery

**GraphQL Platforms:**
- Run introspection query to get exact types
- Scalar types are usually obvious (String, Int, Boolean)
- Custom scalars require testing (e.g., `DateTime`, `Money`)

**REST Platforms:**
```python
def infer_field_type(sample_values):
    """
    Infer type from observed API responses
    """
    if all(isinstance(v, int) for v in sample_values):
        return "integer"
    elif all(isinstance(v, str) and is_iso_datetime(v) for v in sample_values):
        return "datetime"
    elif all(v in [True, False, None] for v in sample_values):
        return "boolean"
    else:
        return "string"  # Default fallback
```

### Data Transformation Detection

**Common Patterns:**

1. **Currency Handling:**
   ```python
   # Test hypothesis: are amounts in cents?
   test_values = [
       (10.00, 1000),   # $10 → 1000
       (15.50, 1550),   # $15.50 → 1550
       (0.99, 99)       # $0.99 → 99
   ]
   ```

2. **Date Format Discovery:**
   ```python
   # Try different formats until one works
   formats = [
       "YYYY-MM-DD",
       "YYYY-MM-DDTHH:mm:ss.sssZ",  # ISO 8601
       "MM/DD/YYYY",
       "DD/MM/YYYY",
       timestamp  # Unix timestamp
   ]
   ```

3. **Enum Value Mapping:**
   ```python
   # UI shows: "Unlimited Stock"
   # Test API values: [null, -1, "UNLIMITED", false (for trackInventory)]
   # Method: Trial and error in sandbox
   ```

### Nested Resource Handling

**Pattern Recognition:**
```
If endpoint is: POST /api/zones/{zoneId}/members
Then structure is: Parent-Child relationship
Inference: Must create parent (zone) before child (members)
```

**Generalized Rule:**
```python
def detect_resource_hierarchy(api_paths):
    """
    Build resource tree from API path patterns
    """
    hierarchy = {}
    
    for path in api_paths:
        segments = path.split('/')
        parent = None
        
        for segment in segments:
            if '{' in segment:  # Path parameter
                hierarchy[parent]['children'].append(segment)
            else:
                hierarchy[segment] = {'parent': parent, 'children': []}
                parent = segment
    
    return hierarchy
```

---

## 4. Validation & Testing Strategy

### Hypothesis-Driven Testing

**Process:**
1. Form hypothesis about API behavior
2. Create minimal test case in sandbox
3. Observe response and system state
4. Confirm or reject hypothesis
5. Update workflow mapping

**Example:**
```
Hypothesis: "Zone ID is required when creating shipping method"

Test:
  mutation { 
    createShippingMethod(input: {
      name: "Test",
      calculator: {...}
      # Omit checker with zoneId
    }) 
  }

Result: ERROR - "Checker is required"
Conclusion: Hypothesis confirmed, checker configuration is mandatory

Updated Hypothesis: "Zone is specified via checker, not direct field"
```

### Boundary Testing

**Test edge cases to understand constraints:**
```python
test_cases = [
    {"name": "", "expected": "ValidationError"},           # Empty string
    {"name": "A" * 300, "expected": "MaxLengthExceeded"},  # Very long string
    {"name": None, "expected": "RequiredFieldError"},      # Null value
    {"name": "<script>", "expected": "Sanitized"},         # XSS attempt
]
```

---

## 5. Application to Other Platforms

### Salesforce Example

**User Intent:** "Create a new Lead for John Doe at Acme Corp"

**Applying the Methodology:**

```
1. API Detection:
   - REST API at /services/data/v58.0/
   
2. Schema Discovery:
   GET /sobjects/Lead/describe/
   
3. Semantic Mapping:
   "Create" → POST /sobjects/Lead/
   "John Doe" → FirstName: "John", LastName: "Doe"
   "Acme Corp" → Company: "Acme Corp"
   
4. Field Discovery:
   Required fields: LastName, Company
   Optional fields: FirstName, Email, Phone, Status
   
5. Workflow Structure:
   {
     "step_1": {
       "api_operation": {
         "method": "POST",
         "endpoint": "/services/data/v58.0/sobjects/Lead/",
         "payload": {
           "FirstName": "John",
           "LastName": "Doe",
           "Company": "Acme Corp"
         }
       }
     }
   }
```

### Jira Example

**User Intent:** "Create a Bug ticket in Project ABC"

```
1. API Detection:
   - REST API v3 at /rest/api/3/
   
2. Schema Discovery:
   GET /rest/api/3/issue/createmeta?projectKeys=ABC
   
3. Semantic Mapping:
   "Create" → POST /rest/api/3/issue/
   "Bug" → issuetype: {id: "10004"}  (via lookup)
   "Project ABC" → project: {key: "ABC"}
   
4. Field Dependencies:
   - Project key → Available issue types
   - Issue type → Required fields for that type
   
5. Multi-step required:
   Step 1: GET project ID from key
   Step 2: GET issue type ID from name
   Step 3: POST issue with resolved IDs
```

### HubSpot Example

**User Intent:** "Add contact with email john@example.com to Marketing list"

```
1. API Detection:
   - REST API at /crm/v3/
   
2. Discovery:
   - Contacts: /crm/v3/objects/contacts
   - Lists: /crm/v3/lists
   
3. Dependency Chain:
   Step 1: Create/find contact by email
   Step 2: Get list ID by name "Marketing"
   Step 3: Add contact to list
   
4. Workflow:
   {
     "step_1": {"operation": "createContact"},
     "step_2": {"operation": "searchLists"},
     "step_3": {
       "operation": "addContactToList",
       "depends_on": ["step_1.contact_id", "step_2.list_id"]
     }
   }
```

---

## 6. Automation-Ready JSON Schema Design

### Design Principles

**1. Machine-Readability:**
Every field must be parsable without human interpretation

**2. Explicit Dependencies:**
No implicit assumptions about execution order

**3. Type Safety:**
All data types explicitly declared

**4. Error Handling:**
Failure scenarios documented for every step

**5. Idempotency:**
Rerunning workflow produces same result

### Schema Structure Rationale

```json
{
  "workflow_metadata": {
    // Enables: Workflow identification, versioning, categorization
  },
  "semantic_mappings": {
    // Enables: Human understanding, reverse mapping from API to UI
  },
  "workflow_steps": [
    {
      "step_id": "unique_identifier",
      "ui_context": {
        // Enables: UI automation (Selenium/Playwright)
      },
      "api_operation": {
        // Enables: Direct API execution (headless)
      },
      "inputs": {
        "required": [],  // Validation before execution
        "optional": []
      },
      "outputs": {
        // Enables: Data extraction for subsequent steps
      },
      "dependencies": {
        "data_flow": []  // Explicit variable binding
      }
    }
  ],
  "data_transformations": {
    // Enables: Reusable transformation logic
  },
  "rollback_strategy": {
    // Enables: Transaction-like behavior
  }
}
```

### Execution Engine Pseudocode

```python
def execute_workflow(workflow_json):
    """
    Generic workflow executor that works with any platform
    """
    steps = topological_sort(workflow_json['workflow_steps'])
    context = {}  # Shared execution context
    
    for step in steps:
        # Resolve dependencies
        inputs = resolve_inputs(step['inputs'], context)
        
        # Execute API call
        response = execute_api_operation(
            step['api_operation'],
            inputs
        )
        
        # Extract outputs
        for output_name, output_spec in step['outputs'].items():
            context[f"{step['step_id']}.{output_name}"] = extract(
                response,
                output_spec['extraction_path']
            )
        
        # Handle errors
        if response.error:
            execute_rollback(workflow_json['rollback_strategy'])
            raise WorkflowExecutionError(step, response.error)
    
    return context
```

---

## 7. Lessons Learned & Best Practices

### Key Insights

1. **Always Start with Schema Introspection**
   - Saves hours of trial-and-error
   - Reveals undocumented features

2. **Network Tab is Ground Truth**
   - Documentation can be outdated
   - Network requests show actual implementation

3. **Test in Sandbox Aggressively**
   - Assumptions are often wrong
   - Edge cases reveal critical constraints

4. **Document Everything**
   - Future you will forget the reasoning
   - Helps team members understand decisions

5. **Think Like a Compiler**
   - User intent → Abstract syntax tree → Execution plan
   - Same principles as code compilation

### Anti-Patterns to Avoid

❌ **Hard-coding values**
```json
{"productId": "123"}  // BAD
```
✅ **Dynamic references**
```json
{"productId": "{step_1.product_id}"}  // GOOD
```

❌ **Ignoring error handling**
```json
{"operation": "createProduct"}  // What if it fails?
```
✅ **Explicit failure scenarios**
```json
{
  "operation": "createProduct",
  "error_handling": {
    "duplicate_name": "append_timestamp_and_retry"
  }
}
```

❌ **Assuming field names**
```json
{"discount": "15%"}  // Is it "discount" or "discountPercent"?
```
✅ **Verified against schema**
```json
{"discountPercent": 15}  // Confirmed via schema
```

---

## Conclusion

This methodology provides a systematic approach to reverse-engineering ANY SaaS platform's workflow automation. The key is treating it as a structured discovery process:

1. **Understand the Intent** (what does the user want?)
2. **Map the UI** (what do they see?)
3. **Discover the API** (what actually happens?)
4. **Bridge the Gap** (how do we translate between them?)
5. **Encode the Knowledge** (how do we make it reusable?)

By following these principles, you can map workflows in Vendure, Salesforce, Jira, HubSpot, or any other platform with the same systematic rigor.

The JSON structure created is not just documentation—it's an executable specification that an automation engine can interpret to perform tasks without human intervention.
