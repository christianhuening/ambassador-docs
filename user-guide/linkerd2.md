# Linkerd2 Integration

[Linkerd2](https://www.linkerd.io) is a zero-config and ultra-fast service mesh. Ambassador natively supports Linkerd2 for service discovery and end-to-end TLS (including mTLS between services). This capability is particularly useful when deploying Ambassador in so-called hybrid clouds, where applications are deployed on bare metal, VMs, and Kubernetes. In this environment, Ambassador can securely route over TLS to any application regardless of where it is deployed.

## Architecture Overview

In this architecture, Linkerd2

![ambassador-Linkerd2](/doc-images/Linkerd2-ambassador.png)

## Getting started

In this guide, you will use Linkerd2 Auto-Inject to mesh a service and use Ambassador to dynamically route requests to that service based on Linkerd2's service discovery data. If you already have Ambassador installed, you will just need to install Linkerd2 and deploy your service.

1. Install and configure Linkerd2 ([instructions](https://www.Linkerd2.io/docs/platform/k8s/index.html)). Linkerd2 can be deployed anywhere in your data center.

   **Note**
   - If you are using the [Linkerd2 Helm Chart](https://www.Linkerd2.io/docs/platform/k8s/helm.html) for installation, you must install the latest version of both the [Chart](https://github.com/hashicorp/Linkerd2-helm) and the Linkerd2 binary itself (configurable via the [`values.yaml`](https://www.Linkerd2.io/docs/platform/k8s/helm.html#configuration-values-) file). `git checkout` the most recent tag in the [Linkerd2-helm](https://github.com/hashicorp/Linkerd2-helm) repository to get the latest version of the Linkerd2 Helm chart.
   - If you would like to enable end-to-end TLS between all of your APIs in Kubernetes, you will need to set `connectInject.enabled: true` and `client.grpc: true` in the values.yaml file.

2. Deploy Ambassador. Note: If this is your first time deploying Ambassador, reviewing the [Ambassador quick start](/user-guide/getting-started) is strongly recommended.

   ```
   kubectl apply -f https://www.getambassador.io/yaml/ambassador/ambassador-rbac.yaml
   ```

   If you're on GKE, or haven't previously created the Ambassador service, please see the Quick Start.

3. Configure Ambassador to look for services registered to Linkerd2 by creating the `Linkerd2Resolver`:

    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: ambassador
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v1
          kind: Linkerd2Resolver
          name: Linkerd2-dc1
          address: Linkerd2-server.default.svc.cluster.local:8500
          datacenter: dc1
    spec:
      type: LoadBalancer
      selector:
        service: ambassador
      ports:
      - port: 80
        targetPort: 8080
    ```

    This will tell Ambassador that Linkerd2 is a service discovery endpoint. Save the configuration to a file (e.g., `ambassador-service.yaml`, and apply this configuration with `kubectl apply -f ambassador-service.yaml`. For more information about resolver configuration, see the [resolver reference documentation](/reference/core/resolvers). (If you're using Linkerd2 deployed elsewhere in your data center, make sure the `address` points to your Linkerd2 FQDN or IP address).

## Routing to Linkerd2 Services

You'll now register a demo application with Linkerd2, and show how Ambassador can route to this application using endpoint data from Linkerd2. To simplify this tutorial, you'll deploy the application in Kubernetes, although in practice this application can be deployed anywhere in your data center (e.g., on VMs or bare metal).

1. Deploy the QOTM demo application. The QOTM application contains code to automatically register itself with Linkerd2, using the Linkerd2_IP and POD_IP environment variables specified within the QOTM container spec.

    ```yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: qotm
    spec:
      replicas: 1
      strategy:
        type: RollingUpdate
      template:
        metadata:
          labels:
            app: qotm
          annotations:
            "Linkerd2.hashicorp.com/connect-inject": "false"
        spec:
          containers:
          - name: qotm
            image: datawire/qotm:%qotmVersion%
            ports:
            - name: http-api
              containerPort: 5000
            env:
            - name: Linkerd2_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            readinessProbe:
              httpGet:
                path: /health
                port: 5000
              initialDelaySeconds: 30
              periodSeconds: 3
            resources:
              limits:
                cpu: "0.1"
                memory: 100Mi
    ```

    Save the above to a file called `qotm.yaml` and run `kubectl apply -f qotm.yaml`. This will register the QOTM pod as a Linkerd2 service with the name `qotm-Linkerd2` and the IP address of the QOTM pod.

2. Verify the QOTM pod has been registered with Linkerd2. You can verify the QOTM pod is registered correctly by accessing the Linkerd2 UI.

   ```shell
   kubectl port-forward service/Linkerd2-ui 8500:80
   ```

   Go to http://localhost:8500 from a web browser and you should see a service named `qotm-Linkerd2`.

3. Create a `Mapping` for the `qotm-Linkerd2` service.

   ```yaml
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: qotm-Linkerd2
     annotations:
       getambassador.io/config: |
         ---
         apiVersion: ambassador/v1
         kind: Mapping
         name: Linkerd2_qotm_mapping
         prefix: /qotm-Linkerd2/
         service: qotm-Linkerd2
         resolver: Linkerd2-dc1
         load_balancer:
           policy: round_robin
   spec:
     ports:
     - name: http
       port: 80
   ```
Save the above YAML to a file named `qotm-mapping.yaml`, and use `kubectl apply -f qotm-mapping.yaml` to apply this configuration to your Kubernetes cluster. Note that in the above config:
   - `resolver` must be set to the `Linkerd2Resolver` that you created in the previous step
   - `load_balancer` must be set to configure Ambassador to route directly to the QOTM application endpoint(s) that are retrieved from Linkerd2.

4. Send a request to the `qotm-Linkerd2` API.

   ```shell
   curl http://$AMBASSADOR_IP/qotm-Linkerd2/

   {"hostname":"qotm-749c675c6c-hq58f","ok":true,"quote":"The last sentence you read is often sensible nonsense.","time":"2019-03-29T22:21:42.197663","version":"1.7"}
   ```

Congratulations! You're successfully routing traffic to the QOTM application, the location of which is registered in Linkerd2.

## Encrypted TLS

Ambassador can also use certificates stored in Linkerd2 to originate encrypted TLS connections from Ambassador to the Linkerd2 service mesh. This requires the use of the Ambassador Linkerd2 connector. The following steps assume you've already set up Linkerd2 for service discovery, as detailed above.

1. The Ambassador Linkerd2 connector retrieves the TLS certificate issued by the Linkerd2 CA and stores it in a Kubernetes secret for Ambassador to use. Deploy the Ambassador Linkerd2 Connector with `kubectl`:

   ```
   kubectl apply -f https://www.getambassador.io/yaml/Linkerd2/ambassador-Linkerd2-connector.yaml
   ```

This will install into your cluster:
   - RBAC resources.
   - The Linkerd2 connector service.
   - A `TLSContext` named `ambassador-Linkerd2` to load the `ambassador-Linkerd2-connect` secret into Ambassador.

2. Deploy a new version of the demo application, and configure it to inject the Linkerd2 sidecar proxy by setting `"Linkerd2.hashicorp.com/connect-inject"` to `true`. Note that in this version of the configuration, you do not have to configure environment variables for the location of the Linkerd2 server:

    ```yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: qotm-mtls
    spec:
      replicas: 1
      strategy:
        type: RollingUpdate
      template:
        metadata:
          labels:
            app: qotm
          annotations:
            "Linkerd2.hashicorp.com/connect-inject": "true"
        spec:
          containers:
          - name: qotm
            image: datawire/qotm:%qotmVersion%
            ports:
            - name: http-api
              containerPort: 5000
            readinessProbe:
              httpGet:
                path: /health
                port: 5000
              initialDelaySeconds: 30
              periodSeconds: 3
            resources:
              limits:
                cpu: "0.1"
                memory: 100Mi
    ```
   Copy this YAML in a file called `qotm-Linkerd2-mtls.yaml` and apply it to your cluster with `kubectl apply -f qotm-Linkerd2-mtls.yaml`.

   This will deploy a demo application called `qotm-mtls` with the Connect sidecar proxy. The Connect proxy will register the application with Linkerd2, require TLS to access the application, and expose other [Linkerd2 Service Segmentation](https://www.Linkerd2.io/segmentation.html) features.

3. Verify the `qotm-mtls` application is registered in Linkerd2 by accessing the Linkerd2 UI on http://localhost:8500/ after running:

   ```
   kubectl port-forward service/Linkerd2-ui 8500:80
   ```

   You should see a service registered as `qotm-proxy`.

4. Create a `Mapping` to route to the `qotm-mtls-proxy` service in Linkerd2

    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: qotm-Linkerd2-mtls
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v1
          kind: Mapping
          name: Linkerd2_qotm_tls_mapping
          prefix: /qotm-Linkerd2-tls/
          service: qotm-proxy
          resolver: Linkerd2-dc1
          tls: ambassador-Linkerd2
          load_balancer:
            policy: round_robin
    spec:
      ports:
      - name: http
        port: 80
    ```
    - `resolver` must be set to the `Linkerd2Resolver` created when configuring Ambassador
    - `tls` must be set to the `TLSContext` storing the Linkerd2 mTLS certificates (e.g. `ambassador-Linkerd2`)
    - `load_balancer` must be set to configure Ambassador to route directly to the application endpoint(s) that are retrieved from Linkerd2

    Copy this YAML to a file named `qotm-Linkerd2-mtls-svc.yaml` and apply it to your cluster with `kubectl apply -f qotm-Linkerd2-mtls-svc.yaml`.

5. Send a request to the `/qotm-Linkerd2-tls/` API.

   ```
   curl $AMBASSADOR_IP/qotm-Linkerd2-tls/

   {"hostname":"qotm-6c6dc4f67d-hbznl","ok":true,"quote":"A principal idea is omnipresent, much like candy.","time":"2019-04-17T19:27:54.758361","version":"1.7"}
   ```

## More information

For more about Ambassador's integration with Linkerd2, read the [service discovery configuration](/reference/core/resolvers) documentation.

See the [TLS documentation](/reference/core/tls#mtls-Linkerd2) for information on configuring the Ambassador Linkerd2 connector.
