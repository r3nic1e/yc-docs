# Connecting to managed databases from functions

Create a connection to access {{ mpg-full-name }} and {{ mch-full-name }} cluster hosts without configured public access from functions.

## Creating a connection {#create}

{% list tabs %}

- Management console

    1. In the [management console]({{ link-console-main }}), go to the folder where you wish to create a connection.
    1. Open **{{ sf-name }}**.
    1. Go to the **Database connections** tab.
    1. Click **Create connection**.
    1. Enter a connection name and description. Name format:

        {% include [name-format](../../_includes/name-format.md) %}

    1. Indicate the following:
        * Cluster.
        * Database.
        * DB user.
        * User password.

    1. Click **Create**.

{% endlist %}

## Connecting to a database {#connect}

To access DB cluster hosts from a function using the created connection:

* In the function version settings, specify the service account with the `serverless.mdbProxies.user` role. [Assigning a role.](./function-public.md#add-access)
* In advanced cluster settings, enable the **Access from Serverless** option.

## Examples of functions for connecting to databases

The connection ID and the entry point are available on the connection page in the [management console]({{ link-console-main }}).

### {{ mpg-name }}

{% list tabs %}

- Node.js

    ```js
    const pg = require('pg');
    
    module.exports.handler = async function (event, context) {
        let proxyId = "akfiotqh2mhktdhf7o47"; // Connection ID
        let proxyEndpoint = "akfiotqh2mhktdhf7o47.postgresql-proxy.serverless.yandexcloud.net:6432"; // Entry point
        let user = "user1"; // DB user
        console.log(context.token);
        let conString = "postgres://" + user + ":" + context.token.access_token + "@" + proxyEndpoint + "/" + proxyId;
    
        let client = new pg.Client(conString);
        client.connect();
    
        let result = await client.query("SELECT 1");
        return result;
    };
    ```

- Python

    ```py
    import psycopg2
    
    
    def handler(event, context):
        connection = psycopg2.connect(
            database="akfiotqh2mhktdhf7o47", # Connection ID
            user="user1", # DB user
            password=context.token["access_token"],
            host="akfiotqh2mhktdhf7o47.postgresql-proxy.serverless.yandexcloud.net", # Entry point
            port=6432,
            sslmode="require")
        cursor = connection.cursor()
        cursor.execute("SELECT 42;")
        record = cursor.fetchall()
        return record
    ```

- Go

    ```golang
    package main
    
    import (
        "context"
        "database/sql"
        "fmt"
        "github.com/yandex-cloud/go-sdk"
    
        _ "github.com/lib/pq"
    )
    
    const (
        host   = "akfv6p92v4gug2jncub5.postgresql-proxy.serverless.cloud-preprod.yandex.net" // Entry point
        port   = 6432
        user   = "user1" // DB user
        dbname = "akfv6p92v4gug2jncub5" // Connection ID
    )
    
    type Response struct {
        StatusCode int         `json:"statusCode"`
        Body       interface{} `json:"body"`
    }
    
    // Getting an IAM token for the service account specified in the function settings
    func getToken(ctx context.Context) string {
        resp, err := ycsdk.InstanceServiceAccount().IAMToken(ctx)
        if err != nil {
            panic(err)
        }
        return resp.IamToken
    }
    
    // Connecting to a database
    func Handler(ctx context.Context) (*Response, error) {
        psqlInfo := fmt.Sprintf("host=%s port=%d user=%s "+"password=%s dbname=%s sslmode=require",
            host, port, user, getToken(ctx), dbname)
        db, err := sql.Open("postgres", psqlInfo)
        if err != nil {
            panic(err)
        }
        defer db.Close()
    
        err = db.Ping()
        if err != nil {
            panic(err)
        }
    
        _, err = db.Query("select 1")
        if err != nil {
            panic(err)
        }
    
        return &Response{
            StatusCode: 200,
            Body:       "Successfully connected!",
        }, nil
    }
    ```

{% endlist %}

### {{ mch-name }}

{% list tabs %}

- Node.js

    ```js
    module.exports.handler = async function (event, context) {
        const https = require('https');
        const querystring = require('querystring');
        const fs = require('fs');
    
        const DB_HOST = "akfd3bhqk359oltp8ejc.clickhouse-proxy.serverless.yandexcloud.net"; // Entry point
        const DB_NAME = "akfd3bhqk359oltp8ejc"; // Connection ID
        const DB_USER = "user1"; // DB user
        const DB_PASS = context.token.access_token;
    
        const CACERT = "/etc/ssl/certs/ca-certificates.crt";
    
        const options = {
            'method': 'GET',
            'ca': fs.readFileSync(CACERT),
            'path': '/?' + querystring.stringify({
                'database': DB_NAME,
                'query': 'SELECT version()',
            }),
            'port': 8443,
            'hostname': DB_HOST,
            'headers': {
                'X-ClickHouse-User': DB_USER,
                'X-ClickHouse-Key': DB_PASS,
            },
        };
    
        return {
            statusCode: 200,
            body: await new Promise((resolve) => {
                data = ''
                const req = https.request(options, (res) => {
                   res.setEncoding('utf8');
                   res.on('data', (chunk) => {
                     data += chunk;
                   });
                   res.on('end', () => { resolve(data) });
                });
                req.end();
            }),
        };
    };
    ```

- Python

    ```py
    import requests
    
    
    def handler(event, context):
        url = 'https://{host}:8443/?database={db}&query={query}'.format(
            host='akfd3bhqk359oltp8ejc.clickhouse-proxy.serverless.yandexcloud.net', # Entry point
            db='akfd3bhqk359oltp8ejc', # Connection ID
            query='SELECT version()')
    
        auth = {
            'X-ClickHouse-User': 'user1', # DB user
            'X-ClickHouse-Key': context.token["access_token"],
        }
    
        cacert = '/etc/ssl/certs/ca-certificates.crt'
    
        rs = requests.get(url, headers=auth, verify=cacert)
        rs.raise_for_status()
    
        return {
            'statusCode': 200,
            'body': rs.text,
        }
    ```

- Go

    ```golang
    package main
    
    import (
        "context"
      "fmt"
      "net/http"
      "io/ioutil"
      "crypto/x509"
      "crypto/tls"
    
      ycsdk "github.com/yandex-cloud/go-sdk"
    )
    
    type Response struct {
        StatusCode int         `json:"statusCode"`
        Body       interface{} `json:"body"`
    }
    
    // Getting an IAM token for the service account specified in the function settings
    func getToken(ctx context.Context) string {
        resp, err := ycsdk.InstanceServiceAccount().IAMToken(ctx)
        if err != nil {
            panic(err)
        }
        return resp.IamToken
    }
    
    // Connecting to a database
    func Handler(ctx context.Context) (*Response, error) {
      const DB_HOST =  "akfd3bhqk359oltp8ejc.clickhouse-proxy.serverless.yandexcloud.net" // Entry point
      const DB_NAME = "akfd3bhqk359oltp8ejc" // Connection ID
      const DB_USER = "user1" // DB user
      DB_PASS := getToken(ctx)
    
      caCertPool, _ := x509.SystemCertPool()
      conn := &http.Client{
        Transport: &http.Transport{
          TLSClientConfig: &tls.Config{ RootCAs: caCertPool },
        },
      }
    
      req, _ := http.NewRequest("GET", fmt.Sprintf("https://%s:8443/", DB_HOST), nil)
      query := req.URL.Query()
      query.Add("database", DB_NAME)
      query.Add("query", "SELECT version()")
    
      req.URL.RawQuery = query.Encode()
    
      req.Header.Add("X-ClickHouse-User", DB_USER)
      req.Header.Add("X-ClickHouse-Key", DB_PASS)
    
      resp, err := conn.Do(req)
      if err != nil {
        if resp != nil {
          data, _ := ioutil.ReadAll(resp.Body)
          panic(data)
        }
        panic(err)
      }
      defer resp.Body.Close()
    
      data, err := ioutil.ReadAll(resp.Body)
      if err != nil {
        panic(err)
      }
    
        return &Response{
            StatusCode: 200,
            Body:       string(data),
      }, nil
    }
    ```
