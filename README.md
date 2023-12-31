# 🏭 High-level architecture overview

## Problem:

Requesting data from an API or database can be expensive. Caching responses can help save time & aid in staying under API rate limits. Using Redis as a proxy between a user/client and a secondary service helps to solve this problem.

## System components

### Redis Proxy - Connect to a pool of Redis connections
    
- Configurable variables:

    - host address, port, database, ttl expiry, cache_capacity, max clients, max memory allocated for the cache, eviction policy
    
- The class is a singleton, to ensure it is a single-backing instance.

- Creates Redis intance with a pool of connections. 

    - implemented features:

        - Redis GET : if key is not saved in the Redis cache, it gets the data from the user-inputed url, saves the response content to Redis and returns the data as a string. If the key is saved in the cache, it returns the data as a string.

### HTTP Server

- Configurable variables:

    - server_class, host address, port

- Maps a HTTP GET request to Redis GET using do_GET()

## Component interacton

RedisProxy can be used on it's own or if the HTTP server is running, with a HTTP GET request. 

Or, RedisProxy can be used via the http_server.py which maps HTTP GET calls to the RedisProxy GET.

## Data Flow

- When a GET request is made, Redis checks its cache for the given key. If it does not exist, a request is made to the given url and saved in Redis the cache with the key.

- Data is input to the system when a key does not exist in the cache and a valid http request can be made. The data from the http request in then saved to the Redis cache.

- Data is output from the system anytime a valid GET request is made. Where the data comes either from the Redis cache or the http response.

### BASIC LOGIC:

    if Redis CONTAINS key:

        GET data and return
    
    else:
    
        GET data from URL
    
        SETEX data to Redis cache with TTL
    
        return data

### Risks

1. I did not implement any security features. AUTH for both Redis and the HTTP server still need to be done.

2. Besides limiting the number of clients, there is no restriction on the number of requests a client can make, so the current process is at risk of outside attacks or misuse.

### Challenges

1. I struggled to figure out a few of the Redis features, especially setting the maxmemory & eviction policy. Sometimes the proxy still outputs a OutOfMemory error when it tries to execute a SETEX command. I would need to do more reserach into handling this error, outside of just allocating more memory to the proxy.

2. I would have liked to write more tests, to catch more of the edge cases, but ran out of time.

# ☝️ What the code does.

redis_proxy.py

    Create connection to Redis and handle functionality of GET

http_server.py

    Creates the HTTP server and maps HTTP GET to Redis GET

load_env.py

    Loads all of the env variables and stores them for other classes to access

logger.py

    Creates a logging object with basic configuration. Can config the logger with the inputs: file_name, class_name, level

test.py

    Runs tests for redis_proxy & http_server. Tests the following feateres: max concurrent users, fixed key size, global expiry, Redis GET, single backing instance, http GET, lru eviction



# 🕐 Algorithmic complexity of the cache operations.

## Time complexity for Redis library commands: 
    
    GET : O(1)
    TTL : O(1)
    SETEX : O(1)
    CONFIG SET : O(N) when N is the number of configuration parameters provided


## Time complexity for RedisProxy cache functions:
    
    redis_get() : O(N) when data is of type `bytes`, O(1) otherwise.
    check_key() : O(1)
    __validate_input() : O(N) when N is the number of configuration parameters provided
    __set_config_features() : O(N) when N is the number of configuration parameters provided
    __set_eviction_policy() : O(1)


# 🚀 Instructions for how to run the proxy and tests.
## Run tests 🔍
    tar -xzvf assignment.tar.gz
    cd assignment
    make test

## Run RedisProxy forever 🔄
1. (optional) Edit the .env file with desired specs
  
2. run

       make run

## Run RedisProxy in a python script ✒️📜
1. (optional) Edit the .env file with desired specs

2. import & call get_instance()

        import http_server
        http_server.run(server_class, host, port)
        requests.get("http_server_host:port", 
             params={'url':'some_url.com', 'key':'key_for_redis_cache', 
                    'params':'{'params to pass to get request of some_url.com':'value'}'})

RedisProxy EXAMPLE:

    '''In the terminal'''
    make redis_start

    '''In a .py script'''
    import redis_proxy
    redis_client = redis_proxy.RedisProxy.get_instance()
    redis_client.redis_get(url, key)
    # OR
    redis_client.redis_get(url, key, payload_for_http_call_with_url)

HTTP GET EXAMPLE:

    '''In the terminal'''
    # set env variables:  HTTP_PORT, HTTP_HOST
    make redis_start
    make http_start # to run in background
    curl -X GET -G 'http://HTTP_HOST:HTTP_PORT' -d 'url=https://someurl.com' -d 'key=test_url'
    make shutdown # to shutdown http & Redis server

    '''OR in a .py script'''
    import http_server
    http_server.run(server_class, host, port)
