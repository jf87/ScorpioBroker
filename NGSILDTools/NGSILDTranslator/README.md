# NGSI-LD Translator
The NGSI-LD Translator is a Python based NGSI-LD Context Producer for various HTTP interactions with a JSON based payload.

## Configuration
The Translator can be configured through a JSON configuration file. This contains 3 main topics

### Source:
The Translator supports 3 types of sources. It can act as datasink for a system which sends notifications. It can continously poll an HTTP source and it can read a file.
This is set by the type parameter which can be:
 - subscribe
 - pull
 - importFromFile
General (optional) configuration options are:
 - isList, which indicates if there is a list of entries incomming per call or if each call gives a single entry.
 - contentIndex, this can be used to shift the "root" for the translation. E.g. in GeoJSON the actual values are in the features entry so you might set the contentIndex to features to simplify the translation part. Dots can be used to subindex in this entry. 
Depending on the type different settings are expected:
 - subscribe:
   - callback:
     - port, to which port should the notifcation callback/datasink listen
	 - hostname, to which hostname should the callback listen. (0.0.0.0 if you want it to listen to all names/ips on the machine)
   - subscribe: this is optional to start a subscription to a remote HTTP system. If provided the translator will attempt to subscribe once on startup.
     - host
     - method, HTTP Method
     - body
     - headers, HTTP Headers for the request
 - pull:
   - from, URL to call 
   - fromHeaders, HTTP Headers for the call
   - pollTime, pause between pulls in ms
 - importFromFile:
   - from, local filesystem place of the file, e.g. "C:\\Users\\hebgen\\Desktop\\al6.GeoJson"

### Translate:
The Translator allows to generically translate JSON inputs with the target to generate a NGSI-LD Entity. In order to achieve this a set of translation possibilities is provided.
General structure:
In general the structure is that the lefthand describes the keys for the NGSI-LD Entity entry and righthand the values based on the input document. For this mapping a set of translation syntax is available. 
 - .(dot) is used to index sublevel of the mapping. level1.level2 equals to a JSON entry of {"level1": {"level2":"somevalue"}}
 - digits between dots are used to index an arrays in an JSON entry. level1.4.level3 is equal to {"level1": ["entry1","entry2","entry3","entry4",{"level3":"fifth value i want"}]}
 - * can only be used on the righthand and has to be followed by another sublevel. It means the first entry in an array which has the sublevel. This can be usefull for handling unsorted arrays where your entry moves randomly with each call.
 - $$ can only be used on the righthand and has to be followed by a dot. It is used to prefix the input data. The following dot indicates the end of the prefix and after it can be considered as root for the input document.
 - &&&& can only be used on the lefthand and has to be at the end of a key and be preceded by a dot. It means that the righthand is the actual value for this entry and not an index for the input document. 
 
In the case you only have one structure in your incomming documents you can put your one mapping into a subentry "*" and it will be used for all incomming documents. 
However there might be the case that you have a varying structure in your incomming documents which require a different mapping but you can give a clear key to identify those. 
In that case you can define an entry "type" on the toplevel of translate. Which is an index of your incomming document. So e.g. 
```json
{"translate": {
   "type": "typeId",
   "232": {... translation...},
   "234": {... translation...}
}
```
Means in your incomming document the Translator checks the value of the field typeId and if it's 232 the first translation will be used and if it's 234 the second.
If you want a fallback to be used you can use "_". So e.g.
```json
{"translate": {
   "type": "typeId",
   "232": {... translation...},
   "234": {... translation...},
   "_": {... fallback translation...},
}
```

### NGSI-LD endpoint:

These are two toplevel entries. Which are ldHost and ldHeaders. ldHeaders should contain at least Content-Type to either application/json or application/ld+json.

### Example
The following example shows mapping a GeoJSON document to an NGSI-LD Entity. 
Translator config:
```json
{
	"source": {
		"type": "pull",
		"from": "http://somehostforgeodata/",
		"fromHeaders": {
			"Accept": "application/json",
			"Content-Type": "application/json"
		},
		"isList": true,
		"pollTime": 36000,
		"contentIndex": "features"
	},
	"translate": {
		"*": {
			"id": "$$urn:smartcity:heidelberg:tunel:.id",
			"type.&&&&": "https://uri.fiware.org/ns/data-models#tunnel",
			"location.type.&&&&": "GeoProperty",
			"location.value.coordinates.0": "geometry.coordinates.0",
			"location.value.coordinates.1": "geometry.coordinates.1",
			"location.value.type": "geometry.type",
			"maximumAllowedWidth.value": "properties.maxwidth",
			"maximumAllowedWidth.type.&&&&": "Property",
			"@context.0.&&&&": "https://schema.lab.fiware.org/ld/context.jsonld"
		}
	},
	"to": "http://localhost:9090",
	"toHeaders": {
		"Content-Type": "Application/ld+json"
	}
}
```
Example data:
```json
{
	"type": "FeatureCollection",
	"features": [{
		"type": "Feature",
		"id": "tunnels.0",
		"geometry": {
			"type": "Point",
			"coordinates": [8.4613486, 49.4935274]
		},
		"geometry_name": "geom",
		"properties": {
			"maxspeed": null,
			"name": "Irgendeinestraße",
			"maxwidth": "3m"
		}
	},
	.......
	]
}
```
As you can see we are only interested in the features entry so the contentIndex is set to features. As the features are an array isList is set to true and since there is only one structure translate has one subentry "*".
Since we use the FIWARE schema in this example the @context arrays first entry is statically set to the url of the schema as well as the type is set to https://uri.fiware.org/ns/data-models#tunnel

This generates an Entity which looks like this
```json
{
	"id": "urn:smartcity:heidelberg:tunnel.0",
	"type": "https://uri.fiware.org/ns/data-models#tunnel",
	"location": {
		"value": [8.4613486, 49.4935274]
	},
	"maximumAllowedWidth": {
		"type": "Property",
		"value": "3m"
	},
	"@context": ["https://schema.lab.fiware.org/ld/context.jsonld"]
}
```
## Starting
To start the Translator run it with 'python translator.py <configfile>'