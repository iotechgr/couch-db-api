# CouchDB API Docs
You can use the HTTP API of CouchDB to acquire data from your IoT devices in your LoRaWAN network that are already stored. 

## CouchDB Views
- Data By EUI (Device Unique identifier)
  
    ```http(s)://{user}:{password}@{host}/{database}/_design/by_eui/_view/by_eui?&descending=true&key=["{eui}"]```

- Data By EUI (Device Unique identifier) using time range
      
    ```http(s)://{user}:{password}@{host}/{database}/_design/by_eui/_view/by_eui_time?&descending=true&startkey=["{eui}","{end_time}"]&endkey=["{eui}","{start_time}"]```

- Data By LRR (Base Station identifier) using time range
    
    ```http(s)://{user}:{password}@{host}/{database}/_design/by_lrr/_view/by_eui_time?&descending=true&startkey=["{lrrid}","{end_time}"]&endkey=["{lrrid}","{start_time}"]```