# REST API Pagination to ADLS

## Overview

This repository demonstrates how to **consume data from a REST API using Azure Data Factory (ADF)** and handle API pagination when the API provides data using **offset and limit query parameters**.

In this implementation:

* **Web Activity** is used to interact with the REST API and retrieve the available data/count information.
* **Copy Activity** is used to fetch the paginated API data.
* Dynamic **offset** and **limit** query parameters are generated to loop through the available records.
* Pagination is configured using the **Query Parameters** pagination rule.
* The retrieved API data is finally stored in **Azure Data Lake Storage (ADLS)** in JSON format.

---

## API Used

The project uses the **PokéAPI** as the REST API source.

Example API request:

```text
https://pokeapi.co/api/v2/pokemon?offset=40?limit=20
```

The API uses:

* `offset` → Starting position of the records to retrieve
* `limit` → Number of records to retrieve per API request

For example:

```text
offset=0?limit=20
offset=20?limit=20
offset=40?limit=20
offset=60?limit=20
...
```

---

## Architecture

```text
              REST API
                 │
                 ▼
        ┌─────────────────┐
        │   Web Activity  │
        │                 │
        │ Retrieve API   │
        │ information    │
        └────────┬────────┘
                 │
                 ▼
       Determine total records
                 │
                 ▼
        Generate Offset Values
        range(0, count, 20)
                 │
                 ▼
        ┌─────────────────┐
        │   Copy Activity │
        │                 │
        │ REST API Source │
        │ + Pagination    │
        └────────┬────────┘
                 │
                 ▼
        ┌─────────────────┐
        │      ADLS       │
        │                 │
        │   JSON Files    │
        └─────────────────┘
```

---

## Pagination Approach

The API returns data in pages, with each request controlled by `offset` and `limit`.

For example, if the API contains **100 records** and the API limit is **20**, the pipeline generates:

```text
range(0, 100, 20)
```

This produces the following offset values:

```text
0
20
40
60
80
```

The `limit` remains:

```text
20
```

Therefore, the pipeline dynamically creates requests such as:

```text
offset=0&limit=20
offset=20&limit=20
offset=40&limit=20
offset=60&limit=20
offset=80&limit=20
```

This allows the pipeline to retrieve the complete dataset instead of only the first page.

---

## Dynamic Query Parameters

The pagination values are dynamically generated rather than hard-coded.

The offset is generated using a range expression based on the total record count:

```text
range(0, <total_count>, 20)
```

The value `20` represents the page size supported by the API.

The dynamically generated offset is then passed into the REST API request along with the limit.

Conceptually:

```text
Offset  → range(0, total_count, 20)
Limit   → 20
```

This makes the pipeline reusable when the total number of records changes.

---

## ADF Pagination Rule

The **Copy Activity** uses the REST API source with the **Query Parameters** pagination rule.

Pagination configuration consists of two important parts:

### Name

The **Name** specifies where the pagination information needs to be injected.

For this API, the query parameter is:

```text
offset
```

### Value

The **Value** specifies where the next-page information comes from.

In this implementation, the value is dynamically generated from the offset range.

Conceptually:

```text
Name  → offset
Value → Dynamic offset generated from range()
```

The `limit` query parameter is also dynamically supplied to control the number of records returned per request.

---

## Absolute URL Pagination

ADF can also handle APIs where the response itself provides the URL for the next page.

For example, an API response might contain:

```json
{
    "next": "https://api.example.com/data?page=2"
}
```

In this scenario, instead of generating `offset` values manually, the **Absolute URL** pagination approach can be used.

The pipeline can follow the `next` URL returned by the API until there are no more pages.

### Two Common Pagination Approaches

| Pagination Type  | When to Use                                                 |
| ---------------- | ----------------------------------------------------------- |
| Query Parameters | API uses parameters such as `offset`, `limit`, `page`, etc. |
| Absolute URL     | API response provides the complete URL for the next page    |

---

## Data Flow

### 1. Web Activity

The Web Activity communicates with the REST API and retrieves the required API information.

This information can be used to determine the total number of records available for processing.

### 2. Generate Pagination Values

The total count is used to generate the required offset values:

```text
range(0, total_count, 20)
```

### 3. Copy Activity

The Copy Activity connects to the REST API and dynamically passes the pagination parameters.

Example:

```text
offset = 0
limit  = 20
```

Then:

```text
offset = 20
limit  = 20
```

And so on until the complete dataset is retrieved.

### 4. Store Data in ADLS

After retrieving the data from the REST API, the Copy Activity writes the output to **Azure Data Lake Storage (ADLS)** in JSON format.

---

## Key Concepts Demonstrated

* REST API integration with Azure Data Factory
* Web Activity
* Copy Activity
* REST API source
* API pagination
* Offset and limit pagination
* Dynamic query parameters
* ADF expressions
* `range()` function
* Query Parameters pagination rule
* Absolute URL pagination
* Dynamic API URL generation
* Writing REST API data to ADLS
* JSON data storage

---

## Example Pagination

Assuming the API contains **100 records** and supports a maximum page size of **20**:

| Request | Offset | Limit |
| ------- | -----: | ----: |
| 1       |      0 |    20 |
| 2       |     20 |    20 |
| 3       |     40 |    20 |
| 4       |     60 |    20 |
| 5       |     80 |    20 |

The pipeline automatically processes all five requests and stores the resulting data in ADLS.

---

## Technologies Used

* **Azure Data Factory**
* **REST API**
* **Web Activity**
* **Copy Activity**
* **Azure Data Lake Storage (ADLS)**
* **JSON**
* **ADF Dynamic Expressions**
* **API Pagination**

---

## Project Outcome

This project demonstrates how to build a reusable **REST API ingestion pipeline in Azure Data Factory** that can dynamically paginate through a large API dataset and load the complete data into **ADLS as JSON**, rather than retrieving only the first page of results.
