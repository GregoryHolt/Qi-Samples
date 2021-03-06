#Python Samples: Building a Client to make REST API Calls to the Qi Service.

This sample demonstrates how Qi REST APIs are invoked using python.

## Creating a connection

The sample uses `httplib` module to connect a service endpoint. A new connection is opened as follows:

```python
	conn = httplib.HTTPConnection(url-name)
```

* url-name is the service endpoint (Ex: "localhost:12345"). The connection is used by the `QiClient` class.  This class encapsulates the REST API and performs authentication.

## Obtain an Authentication Token

The Qi Service is secured by obtaining tokens from an Azure Active Directory instance. The sample applications are examples of a *confidential client*. Such clients provide a user ID and secret that are authenticated against the directory. The sample code includes several placeholder strings. You must replace these with the authentication-related values you received from OSIsoft. The strings are found at the beginning of `test.py`:

```python
authItems = {'resource' : "RESOURCE-URL",
             'authority' : "AUTHORIZATION-URL/oauth2/token",
             'appId' : "CLIENT-ID",
             'appKey' : "CLIENT-SECRET"}
```

You will need to replace `resource`, `appId`, and `appKey`.  The `authItems` array is passed to the `QiClient` constructor.

The Python sample is unique in that it is using raw OAuth 2 calls to obtain an authentication token.  The other samples use libraries from Microsoft that perform the same service, but such a library is not available for Python at this time.  Since OAuth uses HTTP headers, however, it is not hard to do the work yourself if you understand OAuth.  

During initialization, `QiClient` makes the following calls:

```python
self.__getToken()
if not self.__token:
    return
```

The first call, `__getToken`, makes the OAuth call to the Azure ActiveDirectory instance to get a token, then assigns the value (assuming the application is successfully authenticated) to the member variable `__token`.  Thus, following the call to `__getToken`, QiClient checks to ensure it has an authentication token. Calls would fail in the absence of one, hence the return if no token is found.  This is the code for `__getToken`:

```python
    def __getToken(self):     
        if self.__expiration < (time.time()/100):
            return
            
        response = requests.post(self.__authItems['authority'], 
                                 data = { 'grant_type' : 'client_credentials',
                                         'client_id' : self.__authItems['appId'],
                                            'client_secret' : self.__authItems['appKey'],
                                            'resource' : self.__authItems['resource']
                                            })
        if response.status_code == 200:
            self.__token = response.json()['access_token']
            self.__expiration = response.json()['expires_on']
        else:
            self.__token = ""
            print "Authentication Failure : "+response.reason
```

Authentication tokens do expire, however, so the token should be periodically refreshed. The first lines of `__getToken` check to see if the token requires a new call and returns if it does not.  Otherwise, the request is made specifying the type of credentials offered and the resource for which the token is sought.  The `__getToken` method is called before each REST API call, and the token is passed as part of the headers:

```python
conn.request("POST", self.__streamsBase + '/' + qi_stream.Id + self.__insertMultiple, 
     payload, self.__qi_headers())
```

`__qi_headers`, in turn, looks like this:

```python
def __qi_headers(self):
    return {
        "Authorization" : "bearer %s" % self.__token,
        "Content-type": "application/json", 
        "Accept": "text/plain"
    }
```

Note how the value of the `Authorization` header is the word `bearer`, followed by a space, and followed by the token value itself.

## Creating a Qi type

Qi is capable of storing any data type you care to define.  Each data stream is associated with a Qi type, so that only events conforming to that type can be inserted into the stream.  The first step in Qi programming, then, is to define the types for your tenant.

A Qi type has the following properties:

```python
        self.__Id = ""
        self.__Name = None
        self.__Description = None
        self.__QiTypeCode = self.__qiTypeCodeMap[QiTypeCode.Object]
        self.__Properties = []
```

The Qi type "Id" is the identifier for a particular type. "Name" is string property for user understanding. "Description" is again a string that describes the Qi type. "QiTypeCode" is used to identify what kind of datatypes the Qi type stores. The file *QiTypeCode.py* has a list of datatypes that can be stored in the Qi.

A Qi types can have multiple properties, which are again Qi types with the exception that they describe only a single datatype and do not contain multiple properties. Atleast one of the Qi type has to be a key, determined by a boolean value, which is used to index the values to be stored. For example, time can be used as key in order store time based data. Qi allows the use of non-time indices, and also permits compound indices.

From QiTypeProperty.py:
```python
    def __init__(self):
            self.__Id = ""
            self.__Name = None
            self.__Description = None
            self.__QiType = None
            self.__IsKey = False
```            	

A Qi type can be created by a POST request as follows:

```python
	conn.request("POST", "/qi/types/", QiType, 
						headers = {"QiTenant":"tenant_id",
						"Content-type":"application/json",
						"Accept": "text/plain"})
```

* tenant_id	:	Tenant ID of the customer (Every REST call must have a header with a tenant ID)
* Returns the QiType object in a json format
* If a Qi type with the same Id exists, url path of the existing Qi type is returned
* QiType object is passed in json format


## Creating a Qi Stream

Anything in your process that you wish to measure is a stream in Qi, like a point or tag in a chart.  All you have to do is create a local QiStream instance, give it an id, assign it a type, and submit it to the Qi Service.  You may optionally assign a stream behavior to the stream. The value of the `TypeId` property is the value of the Qi type `Id` property.

```python
    def __init__(self):
        self.__Id = 0
        self.__Name = None
        self.__Description = None
        self.__TypeId = None
```

QiStream can be created by a POST request as follows:

```python
conn.request("POST", "/qi/streams/", QiStream, 
						headers = {"QiTenant":"tenant_id",
						"Content-type":"application/json",
						"Accept": "text/plain"})
```

* QiStream object is passed in json format

## Create and Insert Events into the Stream

A single event is a data point in the Stream. An event object cannot be emtpy and should have atleast the key value of the Qi type for the event. Events are passed in json format.

An event can be created using following POST request:

```python
conn.request("POST", "/qi/streams/" + qi_Stream.Id + , "/data/insertvalue", 
						payload,
						headers = {"QiTenant":"tenant_id",
						"Content-type":"application/json",
						"Accept": "text/plain"})
```

* qi_Stream.Id is the stream ID
* payload is the event object in json format

Inserting multiple values is similar, but the payload has list of events and the url for POST call varies

```python
conn.request("POST", "/qi/streams/" + qi_Stream.Id + , "/data/insertvalues", 
						payload,
						headers = {"QiTenant":"tenant_id",
						"Content-type":"application/json",
						"Accept": "text/plain"})
```

## Retrieve Events

There are many methods that allow for the retrieval of events from a stream.  This sample demonstrates the most basic method of retrieving all the events on a particular time range. In general, the index values must be of the same type as the index assigned in the Qi type.  Compound indices' values are concatenated with a pipe ('|') separator.

```python
 conn.request("GET", "/qi/streams/" + qi_stream.Id + "/data/GetWindowValues?" + params,
 						headers = {"QiTenant":"tenant_id",
						"Content-type":"application/json",
						"Accept": "text/plain"})
```

* params has the starting and ending key value of the window
		Ex: For a time index, params will be "endIndex=<*end_time*>&startIndex=<*start_time*>"

## Update Events

Updating events is hanlded by PUT REST call as follows:

```python
 conn.request("PUT", "/qi/streams/" + qi_stream.Id + "/data/replaceValue", 
                     payload,
                     headers = {"QiTenant":"tenant_id",
					"Content-type":"application/json",
					"Accept": "text/plain"})
```

* payload has the new value for the event correseponding to the key value that is to be updated

Updating multiple events is similar but the payload has an array of event objects and url for POST call varies

```python
 conn.request("PUT", "/qi/streams/" + qi_stream.Id + "/data/replaceValues", 
                     payload,
                     headers = {"QiTenant":"tenant_id",
					"Content-type":"application/json",
					"Accept": "text/plain"})
```
##Stream Behaviors
Recorded values are returned by `GetWindowValues`.  If you want to get a particular range of values and interpolate events at the endpoints of the range, you may use `GetRangeValues`.  The nature of the interpolation performed is determined by the stream behavior assigned to the stream.  if you do not specify one, a linear interpolation is assumed.  This example demonstrates a stepwise interpolation using stream behaviors.  More sophisticated behavior is possible, including the specification of interpolation behavior at the level of individual event type properties.  This is discussed in the [Qi API Reference](https://qi-docs.readthedocs.org/en/latest/Overview/).  First, before changing the stream's retrieval behavior, call `GetRangeValues` specifying a start index value of 1 (between the first and second events in the stream) and calculated values:

```python
foundEvents = client.getRangeValues("WaveStreamPy", "1", 0, 3, False, QiBoundaryType.ExactOrCalculated.value)
```

This gives you a calculated event with linear interpolation at index 1.

Now, we define a new stream behavior object and submit it to the Qi Service:

```python
behaviour = QiStreamBehaviour()
behaviour.Id = "evtStreamStepLeading";
behaviour.Mode = QiStreamMode.StepwiseContinuousLeading.value
behaviour = client.createBehaviour(behaviour)
```

By setting the `Mode` property to `StepwiseContinuousLeading` we ensure that any calculated event will have an interpolated index, but every other property will have the value of the recorded event immediately preceding that index.  Now attach this behavior to the existing stream by setting the `BehaviorId` property of the stream and updating the stream definition in the Qi Service:

```python
evtStream.BehaviourId = behaviour.Id
client.updateStream(evtStream)
```

The sample repeats the call to `GetRangeValues` with the same parameters as before, allowing you to compare the values of the event at `Order=1`.

## Deleting Events

An event at a particular data point can be deleted by passing the *key* value for that data point to following DELETE REST call:

```python
 conn.request("DELETE", "/qi/streams/" + qi_stream.Id + "/data/removevalue?" + params, 
                     headers = {"QiTenant":"tenant_id",
					"Content-type":"application/json",
					"Accept": "text/plain"})
```

Delete can also be done over a window of key value as follows:

```python
 conn.request("DELETE", "/qi/streams/" + qi_stream.Id + "/data/removevalues?" + params, 
                     headers = {"QiTenant":"tenant_id",
					"Content-type":"application/json",
					"Accept": "text/plain"})
```

* params has the starting and ending key value of the window
			Ex: For a time index, params:endIndex=<*end_time*>&startIndex=<*start_time*>

## Cleanup: Deleting Types, Behaviors, and Streams

In order to run the sample again in the future without name collisions, the sample does some cleanup at the end. Deleting streams, stream behaviors, and types can be achieved by a DELETE REST call and passing the corresponding ID

```python
 conn.request("DELETE", "/qi/streams/" + stream_id, 
 					headers = {"QiTenant":"tenant_id",
					"Content-type":"application/json",
					"Accept": "text/plain"})
```

```python
conn.request('DELETE', '/qi/types/' + type_id,
					headers = {"QiTenant":"tenant_id",
					"Content-type":"application/json",
					"Accept": "text/plain"})
```
