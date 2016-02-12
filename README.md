Summary
-------

Haproxy managed with confd. The haproxy configuration is performed by confd
which retrieve the settings from a key/value store like *etcd* or *consul*.
This haproxy image support configurations for http and tcp services. The 
proxied services are connected using docker linking system.

Basic authentication is supported for http services. This allow to protect 
the access to a unsecure service. To do this, run your unsecure service on a
protected network that does not allow access from the outside word. The access 
to this service from the outside word will only be allowed through the proxy 
with basic authentication.


Build the image
---------------

To create this image, execute the following command in the docker-haproxy-confd
folder.

    docker build \
        -t cburki/haproxy-confd
        .


Set the configuration
---------------------

    /haservices/<service_id>/scheme             http | tcp
    /haservices/<service_id>/listen_port        port
    /haservices/<service_id>/hosts/<host_id>    address:port
    /haservices/<service_id>/auth/username
    /haservices/<service_id>/auth/password


Run the image
-------------

When you run the image, the port on which the proxy will listen must be
published. You will also link the proxied services for them to be reachable
by haproxy. I suppose the key/value store is running inside a container and
so it must be linked too. The *-backend* arguement is the key/value store
backend to use (e.g. consul) and the *-node* argument is the url of the backend.

    docker run \
        --name haproxy-confd \
        --link <kv_store>:<store_alias> \
        --link <service_container_name>:<service_alias> \
        -d \
        -p <service_port>:<service_port> \
        cburki/haproxy-confd:latest \
        -backend=<config_store_backend> \
        -node=<config_store_url>


Key/Value stores
----------------

Several key/value stores are supported by confd and you can find them in the 
list [here](https://github.com/kelseyhightower/confd/blob/master/docs/quick-start-guide.md).

A single instance of *etcd* could simply be started using the following command.

    docker run \
        --name etcd \
        -d \
        -p 4001:4001 \
        -h kvstore \
        quay.io/coreos/etcd \
        -listen-client-urls http://0.0.0.0:4001 \
        -advertise-client-urls http://$(hostname -I | awk '{ print $1 }'):4001

The configuration could be set using the following commands.

    etcdctl --peers <ip_address>:4001 set /haservices/test/scheme tcp
    etcdctl --peers <ip_address>:4001 set /haservices/test/listen_port 80
    etcdctl --peers <ip_address>:4001 set /haservices/test/hosts/1 1.2.3.4:80
    etcdctl --peers <ip_address>:4001 set /haservices/test/auth/username <username>
    etcdctl --peers <ip_address>:4001 set /haservices/test/auth/password <secret_password>

If you chose to use *consul* as a key/value store, a single instance could be
started using the following command.

    docker run \
        --name consul \
        -d \
        -p 8500:8500 \
        -h kvstore \
        progrium/consul \
        -server -bootstrap

And the configuration is set using curl.

    curl -X PUT -d 'http' http://<ip_address>:8500/v1/kv/haservices/test/scheme
    curl -X PUT -d '80' http://<ip_address>:8500/v1/kv/haservices/test/listen_port
    curl -X PUT -d '1.2.3.4:80' http://<ip_address>:8500/v1/kv/haservices/test/hosts/1
    curl -X PUT -d '<username>' http://<ip_address>:8500/v1/kv/haservices/test/auth/username
    curl -X PUT -d '<password>' http://<ip_address>:8500/v1/kv/haservices/test/auth/password
