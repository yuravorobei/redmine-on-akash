# Deploying Redmine on Akash DeCloud 

![main](https://github.com/yuravorobei/redmine-on-akash/blob/main/media/main.png)

In this guide, I'll show you how to deploy a Redmine on Akash decentralized cloud.

[**Redmine**](https://www.redmine.org/) is a free and open source, web-based project management and issue tracking tool. It allows users to manage multiple projects and associated subprojects. It features per project wikis and forums, time tracking, and flexible role based access control. It includes a calendar and Gantt charts to aid visual representation of projects and their deadlines. Redmine integrates with various version control systems and includes a repository browser and diff viewer.

[**Akash**](https://akash.network/) is a marketplace for cloud compute resources which is designed to reduce waste, thereby cutting costs for consumers and increasing revenue for providers. Akash DeCloud is a faster, better, and lower cost cloud built for DeFi, decentralized projects, and high growth companies, providing unprecedented scale, flexibility, and price performance. 10x lower in cost, Akash serverless computing platform is compatible with all cloud providers and all applications that run on the cloud.

All actions require you to have some knowledge to work with the Linux console. All actions I perform on a Ubuntu 18.04 operating system. And so we go.

# Installing and preparing Akash
To interact with the Akash system, you need to install it and create a wallet. I will follow the official [Akash Documentation guides](https://docs.akash.network/v/master/).

### Choosing a Network
At any given time, there are a number of different active Akash networks running, each with a different akash version, chain-id, seed hosts, etc… Three networks  are currently available: mainnet, testnet and edgenet. In this guide for the "Akashian Challenge: Phase 3 Start Guide" i'll use the `edgenet` network.

So let’s define the required shell variables:
  ```bash
  $ AKASH_NET="https://raw.githubusercontent.com/ovrclk/net/master/edgenet"
  $ AKASH_VERSION="$(curl -s "$AKASH_NET/version.txt")"
  $ AKASH_CHAIN_ID="$(curl -s "$AKASH_NET/chain-id.txt")"
  $ AKASH_NODE="$(curl -s "$AKASH_NET/rpc-nodes.txt" | shuf -n 1)"
  ```

### Installing the Akash client
There are several ways to install Akash: from the [release page](https://github.com/ovrclk/akash/releases), from the source or via `godownloader`. As for me, the easiest and the most convenient way is to download via `godownloader`:
  ```bash
  $ curl https://raw.githubusercontent.com/ovrclk/akash/master/godownloader.sh | sh -s -- "$AKASH_VERSION"
  ```
And add the akash binaries to your shell PATH. Keep in mind that this method will only work in this terminal instance and after restarting your computer or terminal this PATH addition will be removed. You can read how to set your PATH permanently [on this page](https://stackoverflow.com/questions/14637979/how-to-permanently-set-path-on-linux-unix) if you want:
  ```bash
  $ export PATH="$PATH:./bin"
  ```


### Wallet setting
Let's define additional shell variables. `KEY_NAME` value of your choice, I uses a value of “crypto”:
  ```bash
  $ KEY_NAME="crypto"
  $ KEYRING_BACKEND="test"
  ```
Derive a new key locally:
  ```bash
  $ akash --keyring-backend "$KEYRING_BACKEND" keys add "$KEY_NAME"
  ```

You'll see a response similar to below:
  ```yaml
  - name: crypto
    type: local
    address: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
    pubkey: akashpub1addwnpepqvc7x8r2dyula25ucxtawrt39henydttzddvrw6xz5gvcee4arfp7ppe6md
    mnemonic: ""
    threshold: 0
    pubkeys: []


  **Important** write this mnemonic phrase in a safe place.
  It is the only way to recover your account if you ever forget your password.

  share mammal walnut direct plug drive cruise real pencil random regret chunk use live entire gloom donate require autumn bid brown impact scrap slab
  ```
In the above example, your new Akash address is `akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5`, define a new shell variable with it:
  ```bash
  $ ACCOUNT_ADDRESS="akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5"
  ```

### Funding your account
In this guide I use the edgenet Akash network. Non-mainnet networks will often times have a "faucet" running - a server that will send tokens to your account. You can see the faucet url by running:
  ```bash
  $ curl "$AKASH_NET/faucet-url.txt"

  https://akash.vitwit.com/faucet
  ```
Go to the resulting URL and enter your account address; you should see tokens in your account shortly.
Check your account balance with:
  ```yaml
  $ akash --node "$AKASH_NODE" query bank balances "$ACCOUNT_ADDRESS"

  balances: 
  - amount: "100000000" 
   denom: uakt 
  pagination: 
   next_key: null 
   total: "0"
  ```
As you can see, the balance is non-zero and contains 100000000 uakt.
Okay, now you're ready to deploy the application.



# Deploying Redmine


Make sure you have the following set of variables defined on your shell in the previous step "Installing and preparing Akash":
  * `AKASH_CHAIN_ID` - Chain ID of the Akash network connecting to
  * `AKASH_NODE` - Akash network configuration base URL
  * `KEY_NAME` - The name of the key you will be deploying from
  * `KEYRING_BACKEND` - Keyring backend to use for local keys
  * `ACCOUNT_ADDRESS` - The address of your account

### Creating a deployment configuration
For configuration in Akash uses [Stack Definition Language (SDL)](https://docs.akash.network/v/master/documentation/sdl) similar to Docker Compose files. Deployment services, datacenters, pricing, etc.. are described by a YAML configuration file. These files may end in .yml or .yaml.
Create `deploy.yml` configuration file:
  ```bash
  $ nano deploy.yml
  ```
With following content:  

```yaml
version: "2.0"

services:
  db:
    image: mysql:5.7
    env:
        - MYSQL_ROOT_PASSWORD=12345
        - MYSQL_DATABASE=redmine
    expose:
      - port: 3306
        as: 3306
        to:
          - service: db
  
  redmine:
    image: redmine
    depends-on:
        - db
    env:
        - REDMINE_DB_MYSQL=db
        - REDMINE_DB_USERNAME=root
        - REDMINE_DB_PASSWORD=12345
    expose:
      - port: 3000
        as: 80
        to:
          - global: true

profiles:
  compute:
    db:
      resources:
        cpu:
          units: 0.5
        memory:
          size: 512Mi
        storage:
          size: 1Gi
    
    redmine:
      resources:
        cpu:
          units: 0.5
        memory:
          size: 512Mi
        storage:
          size: 1Gi
          
  placement:
    westcoast:    
      pricing:
        db: 
          denom: uakt
          amount: 5000
        redmine: 
          denom: uakt
          amount: 5000

deployment:
  db:
    westcoast:
      profile: db
      count: 1
  redmine:
    westcoast:
      profile: redmine
      count: 1

```

Here we define two services: `db` for the database instance and `redmine` for the Redmine instance. Official [`mysql`](https://hub.docker.com/_/mysql) and [`redmine`](https://hub.docker.com/_/redmine) images from Docker Hub are used as Docker images. For each service specified its deployment parameters. The `services.web.images` field is the name of needed Docker image. The `services.expose.port` field means the container port to expose. The `services.expose.as` field is the port number to expose the container port as specified.  

For the `db` service, define the `MYSQL_ROOT_PASSWORD` environment variable , which specifies the password for the `root` user to login. And `MYSQL_DATABASE=redmine` variable used for auto create database when the image starts. MySQL runs on port `3306`, so expose it. For the `redmine` service, specify the dependency on the` db` service using `depends-on` parameter and redirect access to the service to port` 80`. Also `redmine` service has defined variables: `REDMINE_DB_MYSQL=db` specifies to use the `db` service created above as a database, `REDMINE_DB_USERNAME=root` and `REDMINE_DB_PASSWORD=12345` simply define the login and password to access this database.

Next section `profiles.compute` is map of named compute profiles. Each profile specifies compute resources to be leased for each service instance uses uses the profile. Both defined profiles having resource requirements of 0.5 vCPUs, 512 megabytes of RAM memory, and 1 gigabytes of storage space available. `cpu.units` represent a virtual CPU share and can be fractional.

`profiles.placement` is map of named datacenter profiles. Each profile specifies required datacenter attributes and pricing configuration for each compute profile that will be used within the datacenter. This defines a profile named `westcoast` having required attribute `pricing` with a max price for the `db` and `redmine` compute profiles of 5000 uakt per block. 

The `deployment` section defines how to deploy the services. It is a mapping of service name to deployment configuration.
Each service to be deployed has an entry in the deployment. This entry is maps datacenter profiles to compute profiles to create a final desired configuration for the resources required for the service.


### Creating the deployment
In this step, you post your deployment, the Akash marketplace matches you with a provider via auction. To deploy on Akash, run:
  ```bash
  $ akash tx deployment create deploy.yml --from $KEY_NAME \
    --node $AKASH_NODE --chain-id $AKASH_CHAIN_ID \
    --keyring-backend $KEYRING_BACKEND -y
  ```
Make sure there are no errors in the command response. The error information will be in the `raw_log` responce field.


### Wait for your lease
You can check the status of your lease by running:
  ```
  $ akash query market lease list --owner $ACCOUNT_ADDRESS \
    --node $AKASH_NODE --state active
  ```
  You should see a response similar to:
```yaml
leases:
- lease_id:
    dseq: "208150"
    gseq: 1
    oseq: 1
    owner: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
    provider: akash15ql9ycjkkxhpc2nxtnf78qqjguwzz8gc4ue7wl
  price:
    amount: "1011"
    denom: uakt
  state: active
pagination:
  next_key: null
  total: "0"
```

From this response we can extract some new required for future referencing shell variables:
  ```bash
  $ PROVIDER="akash15ql9ycjkkxhpc2nxtnf78qqjguwzz8gc4ue7wl"
  $ DSEQ="208150"
  $ OSEQ="1"
  $ GSEQ="1"
  ```


### Uploading manifest
Upload the manifest using the values from above step:
  ```bash
  $ akash provider send-manifest deploy.yml \
    --node $AKASH_NODE --dseq $DSEQ \
    --oseq $OSEQ --gseq $GSEQ \
    --owner $ACCOUNT_ADDRESS --provider $PROVIDER
  ```
Your image is deployed, once you uploaded the manifest. You can retrieve the access details by running the `akash provider lease-status` command:
  ```bash
  $ akash provider lease-status \
    --node $AKASH_NODE --dseq $DSEQ \
    --oseq $OSEQ --gseq $GSEQ \
    --provider $PROVIDER --owner $ACCOUNT_ADDRESS
  ```
You should see a response similar to:
  ```json
  {
    "services": {
      "db": {
        "name": "db",
        "available": 1,
        "total": 1,
        "uris": null,
        "observed-generation": 0,
        "replicas": 0,
        "updated-replicas": 0,
        "ready-replicas": 0,
        "available-replicas": 0
      },
      "redmine": {
        "name": "redmine",
        "available": 1,
        "total": 1,
        "uris": [
          "iq1ljrbqmlf3951j7ci4gq9ev4.provider4.akashdev.net"
        ],
        "observed-generation": 0,
        "replicas": 0,
        "updated-replicas": 0,
        "ready-replicas": 0,
        "available-replicas": 0
      }
    },
    "forwarded-ports": {}
  }
  ```
  
You can access the application by visiting the hostnames mapped to your deployment. In above example, its http://iq1ljrbqmlf3951j7ci4gq9ev4.provider4.akashdev.net. Following the address, make sure that the application works:
![result1](https://github.com/yuravorobei/redmine-on-akash/blob/main/media/deployed.png)

### Service Logs
You can view your application logs to debug issues or watch progress using `akash provider service-logs` command, for example:
```
  $ akash provider service-logs --node "$AKASH_NODE" --owner "$ACCOUNT_ADDRESS" \
  --dseq "$DSEQ" --gseq 1 --oseq $OSEQ --provider "$PROVIDER" \
  --service db \
```

You should see a response similar to this:
```
[db-776d7db7c4-wjwml] 2020-12-11 22:18:25+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 5.7.32-1debian10 started.
[db-776d7db7c4-wjwml] 2020-12-11 22:18:26+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
[db-776d7db7c4-wjwml] 2020-12-11 22:18:26+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 5.7.32-1debian10 started.
[db-776d7db7c4-wjwml] 2020-12-11 22:18:26+00:00 [Note] [Entrypoint]: Initializing database files
[redmine-5b45cf756b-zqtxn] The Gemfile's dependencies are satisfied
[db-776d7db7c4-wjwml] 2020-12-11T22:18:39.334720Z 0 [Note] InnoDB: 32 non-redo rollback segment(s) are active.
[redmine-5b45cf756b-zqtxn] I, [2020-12-11T22:18:57.121654 #26]  INFO -- : Migrating to Setup (1)
[db-776d7db7c4-wjwml] 2020-12-11T22:18:39.335504Z 0 [Note] InnoDB: 5.7.32 started; log sequence number 12617750
[redmine-5b45cf756b-zqtxn] == 1 Setup: migrating =========================================================
[db-776d7db7c4-wjwml] 2020-12-11T22:18:39.335784Z 0 [Note] InnoDB: Loading buffer pool(s) from /var/lib/mysql/ib_buffer_pool
[redmine-5b45cf756b-zqtxn] -- create_table("attachments", {:options=>"ENGINE=InnoDB", :force=>true, :id=>:integer})
[db-776d7db7c4-wjwml] 2020-12-11T22:18:39.336228Z 0 [Note] Plugin 'FEDERATED' is disabled.
[redmine-5b45cf756b-zqtxn]    -> 0.0039s
.............
```

### Close your deployment

When you are done with your application, close the deployment. This will deprovision your container and stop the token transfer. Close deployment using deployment by creating a deployment-close transaction:
  ```shell
  $ akash tx deployment close --node $AKASH_NODE \
    --chain-id $AKASH_CHAIN_ID --dseq $DSEQ \
    --owner $ACCOUNT_ADDRESS --from $KEY_NAME \
    --keyring-backend $KEYRING_BACKEND -y
  ```

Additionally, you can also query the market to check if your lease is closed:
  ```bash
  $ akash query market lease list --owner $ACCOUNT_ADDRESS --node $AKASH_NODE
  ```
You should see a response similar to:
  ```yaml
  - lease_id:
      dseq: "208150"
      gseq: 1
      oseq: 1
      owner: akash1kfd3adu7sgcu2vxd3ucc3pehmrautp25kxenn5
      provider: akash15ql9ycjkkxhpc2nxtnf78qqjguwzz8gc4ue7wl
    price:
      amount: "1011"
      denom: uakt
    state: closed
  pagination:
    next_key: null
    total: "0"
  ```
As you can notice from the above, you lease will be marked `closed`.
