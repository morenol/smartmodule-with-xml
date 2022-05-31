# XML handling in Fluvio SmartModules

In Fluvio, records are just raw bytes, therefore we can fill them with any kind of data. In this demonstration, we will create a smartmodule that handles XML data.

## Connector

First, we need to have a source of data. In this demo we will use the Fluvio http connector to retrieve information from the [public APIs](https://api-portal.tfl.gov.uk/) from [Transport for London](https://tfl.gov.uk/)

Specifically, we will use the API to [get bike points occupancy] in XML format.

To create the connector we need a config file. For this example, the config file `bikepoints.yml` looks like:

```yaml
apiVersion: v1
version: 0.2.1
name: bikepoints
type: http
topic: bikepoints
create_topic: true
direction: source
parameters:
  endpoint: https://api.tfl.gov.uk/Occupancy/BikePoints/BikePoints_1,BikePoints_2,BikePoints_3
  header: Accept:text/xml
  method: GET
  interval: 1800
```

This configuration will create a `http` connector called `bikepoints` that produces to topic `bikepoints` the response body from calling the [get bike points occupancy] each 1800 seconds. Note that for this example we are retrieving information from three bike points and that we are using the header `Accept:text/xml` in order to receive a response with XML format.

[get bike points occupancy]: https://api.tfl.gov.uk/swagger/ui/index.html?url=/swagger/docs/v1#!/Occupancy/Occupancy_GetBikePointsOccupancies

With that file, we can create a connector with the command:

```bash
fluvio connector create -c bikepoints.yml
```

Once that is created, the connector will start producing records to the `bikepoints` topic:

```bash
$ fluvio consume bikepoints -B
Consuming records from the beginning of topic 'bikepoints'
<ArrayOfBikePointOccupancy xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2004/07/Tfl.Api.Presentation.Entities"><BikePointOccupancy><BikesCount>1</BikesCount><EBikesCount>0</EBikesCount><EmptyDocks>18</EmptyDocks><Id>BikePoints_1</Id><Name>River Street , Clerkenwell</Name><StandardBikesCount>0</StandardBikesCount><TotalDocks>19</TotalDocks></BikePointOccupancy><BikePointOccupancy><BikesCount>34</BikesCount><EBikesCount>0</EBikesCount><EmptyDocks>2</EmptyDocks><Id>BikePoints_2</Id><Name>Phillimore Gardens, Kensington</Name><StandardBikesCount>0</StandardBikesCount><TotalDocks>37</TotalDocks></BikePointOccupancy><BikePointOccupancy><BikesCount>28</BikesCount><EBikesCount>0</EBikesCount><EmptyDocks>4</EmptyDocks><Id>BikePoints_3</Id><Name>Christopher Street, Liverpool Street</Name><StandardBikesCount>0</StandardBikesCount><TotalDocks>32</TotalDocks></BikePointOccupancy></ArrayOfBikePointOccupancy>
```

Now, we will create an arraymap smartmodule to process each of the records and split them in a record per bike point and in the JSON format, so for example the record above will be converted into:

```bash
{"BikesCount":1,"EBikesCount":0,"EmptyDocks":18,"Id":"BikePoints_1","Name":"River Street , Clerkenwell","StandardBikesCount":0,"TotalDocks":19}
{"BikesCount":34,"EBikesCount":0,"EmptyDocks":2,"Id":"BikePoints_2","Name":"Phillimore Gardens, Kensington","StandardBikesCount":0,"TotalDocks":37}
{"BikesCount":28,"EBikesCount":0,"EmptyDocks":4,"Id":"BikePoints_3","Name":"Christopher Street, Liverpool Street","StandardBikesCount":0,"TotalDocks":32}
```

In order to start that, we will use the [smartmodule templates](https://github.com/infinyon/fluvio-smartmodule-template) and the [cargo-generate](https://github.com/cargo-generate/cargo-generate) utility to generate the files for the smartmodule project. 

We need to choose a name for the project, decide if we want to accept SmartModule parameter and the type of the SmartModule. In this case, we choose `xml-sm`, no parameters and `array-map`.

```bash
$ cargo generate --git https://github.com/infinyon/fluvio-smartmodule-template
# Install with `cargo install cargo-generate`
⚠️   Unable to load config file: /home/lmm/.cargo/cargo-generate.toml
🤷   Project Name : xml-sm
🔧   Generating template ...
✔ 🤷   Want to use SmartModule parameters? · false
✔ 🤷   Which type of SmartModule would you like? · array-map
[1/7]   Done: .cargo/config.toml
[2/7]   Done: .cargo
[3/7]   Done: .gitignore
[4/7]   Done: Cargo.toml
[5/7]   Done: README.md
[6/7]   Done: src/lib.rs
[7/7]   Done: src
🔧   Moving generated files into: `/home/lmm/infinyon/xml-sm`...
💡   Initializing a fresh Git repository
✨   Done! New project created /home/lmm/infinyon/xml-smartmodule

```

This will produce a template with an array-map. Now, we can start editing our smartmodule to behave the way we want. First, for XML deserialization we will add to the Cargo.toml the `quick-xml` dependency with the `serialize` feature.

```toml
...
quick-xml = { version = "0.23.0", features = ["serialize"] }
...
```

We need to define the structs that will store the information from the records. In particular, for the shape of the data that the Bike occupancy API has, we use this:

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "PascalCase")]
pub struct ArrayOfBikePointOccupancy {
    bike_point_occupancy: Vec<BikePointOccupancy>,
}

#[derive(Serialize, Deserialize)]
#[serde(rename_all = "PascalCase")]
pub struct BikePointOccupancy {
    bikes_count: usize,
    e_bikes_count: usize,
    empty_docks: usize,
    id: String,
    name: String,
    standard_bikes_count: usize,
    total_docks: usize,
}
```

Now we just need to deserialize the records from the topic and then for each `bike_point_occupancy` element, we serialize them as JSON and create separated records. That can be done with:

```rust
#[smartmodule(array_map)]
pub fn array_map(record: &Record) -> Result<Vec<(Option<RecordData>, RecordData)>> {
    // Deserialize XML from record
    let array = quick_xml::de::from_slice::<ArrayOfBikePointOccupancy>(record.value.as_ref())?;

    // Create a Json string for each bike point occupancy
    let strings: Vec<String> = array
        .bike_point_occupancy
        .into_iter()
        .map(|value| serde_json::to_string(&value))
        .collect::<core::result::Result<_, _>>()?;

    // Create one record from each JSON string to send
    let kvs: Vec<(Option<RecordData>, RecordData)> = strings
        .into_iter()
        .map(|s| (None, RecordData::from(s)))
        .collect();
    Ok(kvs)
}
```

Now, we can build this smartmodule, upload to Fluvio and then consume from the topic using this smartmodule.

```bash
$ cargo build --release # For building the smartmodule
$ fluvio sm create bikeoccupancy --wasm-file target/wasm32-unknown-unknown/release/xml_sm.wasm
$ fluvio consume bikepoints -B --array-map bikeoccupancy
Consuming records from the beginning of topic 'bikepoints'
{"BikesCount":1,"EBikesCount":0,"EmptyDocks":18,"Id":"BikePoints_1","Name":"River Street , Clerkenwell","StandardBikesCount":0,"TotalDocks":19}
{"BikesCount":34,"EBikesCount":0,"EmptyDocks":2,"Id":"BikePoints_2","Name":"Phillimore Gardens, Kensington","StandardBikesCount":0,"TotalDocks":37}
{"BikesCount":28,"EBikesCount":0,"EmptyDocks":4,"Id":"BikePoints_3","Name":"Christopher Street, Liverpool Street","StandardBikesCount":0,"TotalDocks":32}
{"BikesCount":1,"EBikesCount":0,"EmptyDocks":18,"Id":"BikePoints_1","Name":"River Street , Clerkenwell","StandardBikesCount":0,"TotalDocks":19}
{"BikesCount":34,"EBikesCount":0,"EmptyDocks":2,"Id":"BikePoints_2","Name":"Phillimore Gardens, Kensington","StandardBikesCount":0,"TotalDocks":37}
{"BikesCount":28,"EBikesCount":0,"EmptyDocks":4,"Id":"BikePoints_3","Name":"Christopher Street, Liverpool Street","StandardBikesCount":0,"TotalDocks":32}
```

That's all! Now you have a smartmodule that handles XML data and does transformation on top of that.