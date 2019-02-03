
# Data Universe API


## MultiverseQL
Data Universe is called in the API using a syntax similar to [GraphQL](graphql.org), extended for use with the data universe needs - *MultiverseQL* (name tbd).

We start with the same simple syntax as GraphQL, based on a root object:

```GraphQL
action {
  context
}
```

There are a few pre-determined actions (this varies from GraphQL where there are only two defined actions `query` and `mutate`).  The actions available can also be chained, as if they were in a pipeline.  The overall context is built up and may become applied to a [Polyform](#polyform) as the Pans(data in the universe) and Dims (runtime dimensions).

* *Polyform* - a polyform is AI as a service, effectively it is the combination of a function run in a specific container context, where the data needed to run the AI is also curated and presented to the container at run time.
* *Pans* - these are items stored in the Data Universe.  All data can be represented by a Pan, and Pans have a unique key mapping system plus key security cryptography to protect it properly.
* *Dims* - dimensions are added at runtime to help the AI understand the full context of how it should run.  Such as who is requesting, what timeframe, etc.  Pans are also brought in as Dims.  The Polyform is the intersection of the Pans plane of existence, and the Dims plane of existence, at which point the function is run.

### Data formatting

Although shown loosely in these examples, the simple types of data are similar to JSON. You can skip this and go straight to (examples)[#examples].

- `"string"` - a string begins and ends with double quotes
- `000` - an Int64 number
- `0.000` - a Float64 number
- `literal` - these are used within our query system, and only support from the set: `[A-Za-z0-9]`
- `attribute.name` - attributes are a more complex system, where literals are combined to define a broader name context. Note: dashes nor underscores are supported.  Only periods for delimiters, colon for separator, and a hash for a selector (when there are multiple sub types of a given element, such as currency as usd, eu, etc).

Because attributes can be relative and include data from the overall building context, the syntax also includes things such as:
- `@namedthing:` - this references `namedthing.id` and it's graphed related items, pulling the key following the colon.
- `account.balance#usd` - this is a selector, where the value is actually account.balance, but it is provided in us dollars.

Grouping of blocks is with braces `{ }`, but context within the block  and grouping of an array is with square braces `[ ]`, comma delimited.

### Actions

The actions available are not required to be in order, other than the first (sig):

- `access` - access token (see below)
- `dims` - dimensions specific to this request.  Consider this like the payload section of a JWT.  This is hashed with a pre-shared secret and the hash value is stored in the `sig` block.  The dims should at a minimum include some values in context:
    - `initiator`.`id` - this maps to the initiator as a pre-defined entity in the universe, to which the pre-shared secret is correlated
    - `

- `new|into` - create pans around the described information.  If new pans are created, their mapIds are given in the result-set
- `gather` - gather additional PAN and values from the system (can call other poly's even)
- `join|union|add as LABEL` - how to bring the data together such that the intersect is expecting things.  The LABEL is used as a key to the function, which points to the data file descriptor for the container to read from.
    - `join` will create a dataframe array, with each of the gathered pans as a column.
    - `union` will create a dictionary, with only one key value for each key name (the order of the pans matter--the latter ones taking precedence
    - `setadd` will create a dataframe array, but only add the pan if it's values are unique within the set (no duplicates)
- `intersect(name:version)` - run an intersect.  The context describes how to adjust the Pans and Dims into the event used by the intersect.
- `into` - This immediately follows an intersect, and it adjusts the results from the intersect, adding context as they are converted into PANs and returned to the session.
- `query` - separate request, specifically calling data
- `transform(type)` - give raw transform using a different serialized format, such as `transform(handler:version)`.  A polyform(handler) is run to parse the input.  The result-set become pans, and the result is a key:value dictionary that may be pulled into this context with `${key}`.  The context for input is defined with keywords (if one of variables, data or multipart is not specified, assumption is variables):
    - `contentType` = `"content type"` - optional
    - `variables` = `true` - pull from the query variables
    - `data` = `"serialized data"` - this or the next
    - `multipart` = - TBD: figure out HTTP2/3 multipart submits for data
    - `encoding` = `"base64"` - for `data` - if undefined assume "none"

### Data Transforms in Flight

You can also transform the data in flight using a pipe operator `|>` and functions thereafter.  Such as:

```GraphQL
this.number.is.a.string = "20000.0" |> toInt
```

This is supported within `new`, `into`, `gather`, `join`, `union`, and `setadd`.

Data transformed in flight is ephemeral until it is committed to a pan (with either a `new` or `into` action).

# Notes:
- cursor to large data sets?  do we want large data set results?

## Examples

### Access Token

An access token is a JWT sent as an HTTP bearer header.

The access token is acquired through a separate action described in [Port Authentication](#Authentication)

The access token includes the claims as shown, and the subject claim expands to import as an `@initiator` dimension.

  - exp - posix seconds (integer) until this signature expires (no more than 5 minutes from now allowed)   
  - sub - a unique key for this access token
  - aud - the expected audience (i.e. the service you are targeting) and the auth has been approved for
  - for - an array of Polyforms that are allowed

Claim Examples (as JSON):

```json
{
  "exp": 1549205284,
  "sub": "3d5b9891-7b23-4c47-8d4a-48e0d886715a",
  "aud": "portal.pandim.xyz:443",
  "for": ["ingress", "transform", "intersect(Decisioning)"]
}
```

### Submit Data

Submit via HTTP Post, example (submitting to a credit decisioning AI):

```GraphQL
dims {
  person.id =   "FACFAF-1FA14D-89FA"
  property.id = "B0GP8E-0094E5-ED11"
  employer.id = "EA9F28-04555A-41D0"
} new {
  @person:fintec.ccard.sales = 20000
  @person:gross.sales = 43904
  @person:dib_ownership = 2740
  @person:dib_establish = 2745
  @employer:employees = 52
}
```

### Transform Data in flight

It may be that you want to send your data inputs in a different format, which can be supported by using a separate transform polyform to process the data.  For example:

```GraphQL
dims {
  person.id =   "FACFAF-1FA14D-89FA"
  property.id = "B0GP8E-0094E5-ED11"
  employer.id = "EA9F28-04555A-41D0"
} transform(jsonHandler:latest) {
  data = "{
    \"ccard-sales\": {\"0\":\"20000.0\"},
    \"gross_sales\": {\"0\":43904},
    \"dib_ownership\": {\"0\":\"2740.0\"},
    \"dib_establish\": {\"0\":\"2745.0416666666665\"},
    \"employees\": {\"0\":52}
  }"
} into {
  @person:fintec.ccard.sales = ${ccard-sales} |> dig("0") |> toInt
  @person:gross.sales = ${gross_sales} |> dig("0") |> toInt
  @person:dib_ownership = ${dib_ownership} |> dig("0") |> toInt
  @person:dib_establish = ${dib_establish} |> dig("0") |> toInt
  @employer:employees = ${employees} |> dig("0") |> toInt
}
```

The primary difference between transform and intercept is that transform runs before all other steps, regardless of where it is placed in the chain, and intercept runs at the point in the chain it is placed.

### Intersecting with a Polyform

An intersection with a polyform is when all of the pans and dimensions come together to run a native AI function.  This is the polyform.

You can chain an intersect immediately after pan creation, but it is not required.

```GraphQL
dims {
  person.id =   "FACFAF-1FA14D-89FA"
  property.id = "B0GP8E-0094E5-ED11"
  employer.id = "EA9F28-04555A-41D0"
} join as "inputs" {
  @person:finsec.ccard.sales
  @person:finsec.bankruptcy.score |> as(score)
  @person:finsec.fico
  @person:finsec.revolving
  @person:gross.sales
  @person:dib_ownership
  @person:dib_establish
  @person:dob |> toAge |> as(age)
  @person:comp.wage#usd |> toInt |> as(wage)
  @employer:employees
  @property:location.sqft
  @employer:unemployment_ranking
  @initiator:trust.of.vendor
} intersect Decisioning(latest) {
} into {
  @person:credit[@initiator] = [
    recommendation.score
    payment.pace#months
    suggested.amount#usd
  ]
} return {
  @result {
    recommendation.score
  }
  @employer:dins
}
```
Note:
* update actions in list above.  `return` notably.  This is like a graphql way of scoping which params you want in the result set.  If you do not include a return, it only returns what is included from the `into` as a limited data set.
* also note: this is how abac kicks in.  I included `@employer:dins` as a way somebody might try to hack and gain more information.  In this case the initiator does not have access to that information, even if the polyform does, and we should decide if it results in a failure, or a smaller result set.  If we created employer:dins as part of this query, then we'd also have access to get it as a result set.

```json
{
  "result": [
    { "recommendation.score": 0.9959492111,
      "pragma": "no-cache",
      "duid": "DAFV-QFES" }
  ],
  "employer": []
}
```
but to query one pan specifically (should this support graphQL querying?)

