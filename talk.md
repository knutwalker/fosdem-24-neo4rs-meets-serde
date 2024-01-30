---
theme: solarized
highlightTheme: css/solarized-light.css
transition: none
slideNumber: true
progress: true
controls: false
timeForPresentation: 2400
---

# The journey of hacking in a new serde dataformat
## 2024-02-03 / FOSDEM '24 / Rust devroom

---

## Abstract (paraphrased)

> [`neo4rs`](https://github.com/neo4j-labs/neo4rs) use serde.
> I want to present the journey of building [that].

---

## What this talk is not

+ Introduction to serde
+ Introduction to neo4rs
+ An actual _deep_ deep-dive
+ Discussion about how to pronounce serde

---

## About me

- Hi, I'm Paul
- `@knutwalker[@hachyderm.io]`

---

<!-- slide bg="[[neo4j-logo.png]]" data-background-size="contain"  -->

---

<!-- slide bg="[[neo4j-logo.png]]" data-background-size="50% auto" data-background-position="50% 10%" -->
## About Neo4j®

- A Graph Database written in Not-Rust
- We have a Rust driver: `neo4rs`, written in pure Rust
- Developed under Neo4j Labs
- [github.com/neo4j-labs/neo4rs](https://github.com/neo4j-labs/neo4rs)

---

## About Neo4j®

- Neo4j drivers generally comunicate via [Bolt](https://7687.org)
 + Bolt is a binary protocol
	+ packstream for general data types
	+ "binary JSON-ish"
	+ Domain-specific structs (~15)

---

## About neo4rs

- Bolt structs as a `BoltType` enum

```rust
enum BoltType {
    Null,
    Integer(i64),
    String(String),
    List(Vec<BoltType>),
    Node(BoltNode),
    // ...
}
```

---

## `neo4rs` 0.6

```rust
let event = node.get::<String>("event").unwrap();
let year = node.get::<u64>("year").unwrap();
```

---

## `neo4rs` 0.7

```rust
let event = node.get::<String>("event").unwrap();
let year = node.get::<u64>("year").unwrap();
```

---

## `neo4rs` 0.7

```rust
#[derive(serde::Deserialize)]
struct Session {
    event: String,
    year: u64,
}
```

---

## `neo4rs` 0.7

```rust
let session = node.to::<Session>().unwrap();
```

---

## About `serde`

- A framework for _**ser**_ializing and _**de**_serializing Rust data structures ([serde.rs](serde.rs))
- [Jon's decrusting video](https://www.youtube.com/watch?v=BI_bHCGRgMY)

---

## About `serde`

- data type
+ `#[derive(Serialize, Deserialize)]`
- data format
+ `Serializer` and `Deserializer` traits
+ notice the **r** at the end of the names
- data model
+ mostly represented in the API only

---

## About `serde`

- example data format: JSON
+ data format implementation: `serde_json`

---

## `neo4rs` meets `serde`

- In `neo4rs`, data is already available as `BoltType`
+ Implement a data format for `BoltType`
+ `Deserializer` only
+ Maintain API compatibility, if possible

---

### Node

```
Node::Structure(
    id::Integer,
    labels::List<String>,
    properties::Dictionary,
)
```

---

### Node

```rust
pub struct BoltNode {
    pub id: BoltInteger,
    pub labels: BoltList,
    pub properties: BoltMap,
}
```

---

### Example

```cypher
CREATE (n:Session { event: 'FOSDEM', year: 2024 })
RETURN n
```

---

### Example

```cypher [1-7|2|3-6|7]
CREATE (n
    :Session
    {
        event: 'FOSDEM',
         year: 2024
    }
) RETURN n
```

---

### Example

```rust
#[derive(Deserialize)]
struct Session {
    event: String,
    year: u64,
}
```

---

### Example

```rust [1|3|4]
let session = row.get::<Session>("n").unwrap();

let node = row.get::<Node>("n").unwrap();
let session = node.to::<Session>().unwrap();
```

---

#### First attempt

```rust [2-6|1]
#[derive(Deserialize, Serialize)]
pub struct BoltNode {
    pub id: BoltInteger,
    pub labels: BoltList,
    pub properties: BoltMap,
}
```

---

#### First attempt

```rust [1-5|1|2|3]
pub fn to<T: Deserialize>(self) -> Result<T, Error> {
    let value = serde_json::to_value(self)?;
    let result = serde_json::from_value(value)?;
    Ok(result)
}
```

---

#### First attempt

Are we done?

```


```

---

#### First attempt

Are we done?

```
missing field `event`
missing field `year`
```

---

#### First attempt

```rust
#[derive(Deserialize, Serialize)]
pub struct BoltNode {
    pub id: BoltInteger,
    pub labels: BoltList,
    pub properties: BoltMap,
}
```

---

#### First attempt

```rust
#[derive(Deserialize)]
struct SessionNode {
    properties: Session
}
```

---

#### Second attempt

```rust [2]
pub fn to<T: Deserialize>(self) -> Result<T, Error> {
    let value = serde_json::to_value(self.properties)?;
    let result = serde_json::from_value(value)?;
    Ok(result)
}
```

---

#### Second attempt

- Kinda works
+ No way to get id and labels

---

#### Third attempt

```rust [1-4,7|5-6]
#[derive(Deserialize)]
struct Session {
    event: String,
    year: u64,
    id: u64,
    labels: Vec<String>,
}
```

---

### Small excursion into `serde`

```rust [1,9|2|3|4,5]
impl Deserialize for Session {
    fn deserialize<D>(deserializer: D)
     -> Result<Self, D::Error>
    where
        D: Deserializer,
    {
        todo!()
    }
}
```

---

### Small excursion into `serde`

```rust [3-5,7|6]
impl Deserialize for Session {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error> {
        deserializer.deserialize_struct(
            "Session",
            &["event", "year", "id", "labels"],
            todo!("Visitor"),
        )
    }
}
```

---

### Small excursion into `serde`

```rust [1|3,9|4|6-8]
struct SessionVisitor;

impl Visitor for SessionVisitor {
    type Value = Session;

    fn expecting(&self, formatter: &mut Formatter) -> fmt::Result {
        Formatter::write_str(formatter, "struct Session")
    }
}
```

---

### Small excursion into `serde`

```rust [2|3|4,5]
impl Visitor for SessionVisitor {
    fn visit_map<A>(self, mut map: A)
     -> Result<Self::Value, A::Error>
    where
        A: MapAccess,
    {
        todo!()
    }
}
```

---

### Small excursion into `serde`

```rust [2-5]
fn visit_map<A: MapAccess>(self, mut map: A) -> Result<Self::Value, A::Error> {
    let mut event: Option<String> = None;
    let mut year: Option<u64> = None;
	let mut id: Option<u64> = None;
	let mut labels: Option<Vec<String>> = None; 
}
```

---

### Small excursion into `serde`

```rust [2|3|4|5|6-8]
fn visit_map<A: MapAccess>(self, mut map: A) -> Result<Self::Value, A::Error> {
    while let Some(key) = map.next_key::<&str>()? {
        match key {
            "event" => event =
	            Some(map.next_value::<String>()?),
            "year" => year = Some(map.next_value::<u64>()?),
            "id" => id = Some(map.next_value::<u64>()?),
            "labels" => labels = Some(map.next_value::<Vec<String>>()?),
            _ => todo!("unknown field"),
        }
    }
}
```

---

### Small excursion into `serde`

```rust [2,9|3-5|6-8]
fn visit_map<A: MapAccess>(self, mut map: A) -> Result<Self::Value, A::Error> {
    Ok(Session {
        event: event.ok_or_else(
	        || Error::missing_field("event")
	    )?,
        year: year.ok_or_else(|| Error::missing_field("year"))?,
		id: id.ok_or_else(|| Error::missing_field("id")),
		labels: labels.ok_or_else(|| Error::missing_field("labels")),
    })
}
```

---

### Small excursion into `serde`

```rust [6]
impl Deserialize for Session {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error> {
        deserializer.deserialize_struct(
            "Session",
            &["event", "year"],
            SessionVisitor,
        )
    }
}
```

---

#### Third attempt

```rust [1|3-9]
struct BoltNodeDeserializer(BoltNode);

impl IntoDeserializer<DeError> for BoltNode {
    type Deserializer = BoltNodeDeserializer;

    fn into_deserializer(self) -> Self::Deserializer {
        BoltNodeDeserializer::new(self)
    }
}
```

---

#### Third attempt

```rust [1-2|4-9]
impl Deserializer for BoltNodeDeserializer {
    type Error = DeError;

    forward_to_deserialize_any! {
        bool i8 i16 i32 i64 i128 u8 u16 u32 u64 u128
        f32 f64 char str string bytes byte_buf option
        unit unit_struct seq tuple tuple_struct struct
        identifier enum map ignored_any newtype_struct
    }
}
```

---

#### Third attempt

```rust [2|3|4-5|7,9|8]
impl Deserializer for BoltNodeDeserializer {
    fn deserialize_any<V>(self, visitor: V)
     -> Result<V::Value, Self::Error>
    where
        V: Visitor,
    {
        visitor.visit_map(MapDeserializer::new(
            self.0.properties.iter()
        ))
    }
}
```

---

#### Third attempt

```rust [2]
fn to<T: Deserialize>(self) -> Result<T, DeError> {
    T::deserialize(self.into_deserializer())
}
```

---

#### Third attempt

- We now have the same result as before
+ Let's add `id` and `labels`

---

#### Third attempt

```rust [4|5,8|6|7|9]
impl Deserializer for BoltNodeDeserializer {
    fn deserialize_any<V>(self, visitor: V)
     -> Result<V::Value, Self::Error> {
        let properties = self.0.properties.iter();
        let properties = properties.chain([
            ("id", &self.0.id),
            ("labels", &self.0.labels),
        ]);
        visitor.visit_map(MapDeserializer::new(properties))
    }
}
```

---

#### Third attempt

- Kinda works
+ We do get id and labels, but:
+ We only map them by the names `id` and `labels`
+ We could use something like `__id`, but, meh
+ Let's try something else instead

---

#### Fourth attempt

```rust
struct Id(pub u64);
struct Labels(pub Vec<String>);
```

---

#### Fourth attempt

```rust [1-4,7|5-6]
#[derive(Deserialize)]
struct Session {
    event: String,
    year: u64,
    id: neo4rs::Id,
    labels: neo4rs::Labels,
}
```

---

#### Fourth attempt

```rust [2-7|4-5]
impl Deserializer for BoltNodeDeserializer {
    fn deserialize_struct<V>(
        self,
        _name: &'static str,
        fields: &'static [&'static str],
        visitor: V,
    ) -> Result<V::Value, Self::Error>
    where
        V: Visitor,
    {
        visitor.visit_map(MapDeserializer::new(todo!()))
    }
}
```

---

#### Fourth attempt

```rust [3-5|7-9|10|11]
impl Deserializer for BoltNodeDeserializer {
    fn deserialize_struct<V>(self, fields: &[&str], visitor: V) -> Result<V::Value, Self::Error> {
        let property_fields = self.0.properties
            .iter()
            .map(|(k, v)| (k, StructData::Property(v)));

        let additional_fields = fields
            .iter()
            .copied()
            .filter(|f| !self.0.properties.contains_key(*f))
            .map(|f| (f, StructData::Node(self.0)));
    }
}
```

---

#### Fourth attempt

```rust [4-5|7]
impl Deserializer for BoltNodeDeserializer {
    fn deserialize_struct<V>(self, fields: &[&str], visitor: V) -> Result<V::Value, Self::Error> {
        // ...
        let node_fields =
            property_fields.chain(additional_fields);

        visitor.visit_map(MapDeserializer::new(node_fields))
    }
}
```

---

#### Fourth attempt

```rust [2,3,11|4]
impl Deserializer for AdditionalNodeData {
    fn deserialize_newtype_struct<V>(
	    self,
	    name: &str,
	    visitor: V
	) -> Result<V::Value, Self::Error>
    where
	    V: Visitor,
    {
		todo!()
    }
}
```

---

#### Fourth attempt

```rust [3|4-6|7-11]
impl Deserializer for AdditionalNodeData {
    fn deserialize_newtype_struct<V: Visitor>(self, name: &str, visitor: V) -> Result<V::Value, Self::Error> {
        match name {
            "Id" => visitor.visit_newtype_struct(
	            self.data.id.into_deserializer()
            ),
            "Labels" => visitor.visit_newtype_struct(
	            SeqDeserializer::new(
					self.data.labels.iter()
		        )
		    ),
            _ => todo!()
        }
    }
}
```

---

#### Fourth attempt

- Works
+ There are still some downsides
+ `#[serde(default)]` doesn't work

---

### `#[serde(default)]`

```rust [9|5]
fn visit_map<A: MapAccess>(self, mut map: A) -> Result<Self::Value, A::Error> {
    let mut year: Option<u64> = None;
    while let Some(key) = map.next_key::<&str>()? {
        match key {
            "year" => year = Some(map.next_value::<u64>()?),
        }
    }
    Ok(Session {
        year: year.unwrap_or_else(|| Default::default())?,
    })
}
```

---

### `#[serde(default)]`

```rust [3-7]
impl Deserializer for BoltNodeDeserializer {
    fn deserialize_struct<V>(self, fields: &[&str], visitor: V) -> Result<V::Value, Self::Error> {
        let additional_fields = fields
            .iter()
            .copied()
            .filter(|f| !self.0.properties.contains_key(*f))
            .map(|f| (f, StructData::Node(self.0)));
    }
}
```

---

#### Fourth attempt

- Works
- There are still some downsides
- `#[serde(default)]` doesn't work
+ Workaround: `Option<T>`

---

### `BoltType`

- Rinse and repeat for `BoltType` and its ~20 variants
+ Almost...

---

### `BoltType::Bytes`

```rust [2|3-9|6-7]
impl Deserializer for BoltTypeDeserializer {
    fn deserialize_any<V>(self, visitor: V) -> Result<V::Value, Self::Error> {
        match self.value {
            BoltType::String(v) =>
		        visitor.visit_string(v.clone()),
            BoltType::Bytes(v) =>
	            visitor.visit_bytes(v.clone()),
            // ...
        }
    }
}
```

---

### `BoltType::Bytes`

```rust [4]
impl Deserializer for BoltTypeDeserializer {
    fn deserialize_any<V>(self, visitor: V) -> Result<V::Value, Self::Error> {
        match self.value {
            BoltType::Bytes(v) => visitor.visit_seq(SeqDeserializer(v.iter())),
            // ...
        }
    }
}
```

---

### `BoltType::Bytes`

```rust [2-4|6|7]
impl Deserializer for BoltTypeDeserializer {
    fn deserialize_bytes<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor,
    {
        if let BoltType::Bytes(v) = self.value {
            visitor.visit_bytes(v.clone())
        } else {
            self.unexpected(visitor)
        }
    }
}
```

---

### Are we done now?

- No, but out of time 😅
+ Here are more things to consider:
+ Maintain existing API:
    + Have special cases for converting into chrono/time types
        + `is_human_readable`
        + different precision for timestamps

---

### Are we done now?

- No, but out of time 😅
- Here are more things to consider:
- Maintain existing API:
    + Need to deserialize into itself
        + Custom `Deserialize` impl for `BoltType`
        + That impl is not really usable for other data formats
---

### Are we done now?

- No, but out of time 😅
- Here are more things to consider:
- Allow for unexpected fields
    + `deserialize_ignored_any`

---

### Are we done now?

- No, but out of time 😅
- Here are more things to consider:
- Allow for zero-copy deserialization
    + Keep `'de` lifetimes around
    + implement all the `_borrowed` methods

---

### Thank you!
#### Q/A?
## `use std::process::exit;`
# `exit(42);`

