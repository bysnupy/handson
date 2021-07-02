# What metrics are used in Console Overview Status on OCPv4

![ocp4 cross projects ingress](https://github.com/bysnupy/handson/blob/master/ocp4__console_health_status.png)

##1 "Cluster" is based on the k8s API[0][1]. 
   At the console, the cluster health state is fetched by "/api/kubernetes/healthz" and it make proxy internally as follows[2].

  https://console-openshift-console.apps<cluster name>.<base domain>/api/kubernetes/healthz
  --proxy by console internally--> https://<k8s base url>/healthz


  [0] Kubernetes API health endpoints
      [https://kubernetes.io/docs/reference/using-api/health-checks/]

  [1] console/frontend/packages/console-app/src/plugin.tsx
  ```ts
    type: 'Dashboards/Overview/Health/URL',
    properties: {
      title: 'Cluster',                     <--- Health card name
      url: 'healthz',                       <--- API endpoint
      fetch: fetchK8sHealth,
      healthHandler: getK8sHealthState,
      :
  ```

  [2] OpenShift Console
      [https://github.com/openshift/console#openshift-console]
  ```
  * Proxying the Kubernetes API under /api/kubernetes
  ```

##2 "Control Plane" depends on Prometheus metrics, refer the query details are in [3] and [4].
   You can test those queries at the web console --> Monitoring --> Metrics --> Copy and pate each query to the expression text box and Run the query.

  [3] console/frontend/packages/console-app/src/plugin.tsx
  ```ts
    type: 'Dashboards/Overview/Health/Prometheus',
    properties: {
      title: 'Control Plane',                                                                         <--- Health card name
      queries: [API_SERVERS_UP, CONTROLLER_MANAGERS_UP, SCHEDULERS_UP, API_SERVER_REQUESTS_SUCCESS],  <--- prometheus query url
      healthHandler: getControlPlaneHealth,
      :
  ```

  [4] console/frontend/packages/console-app/src/queries.ts
  ```ts
  export const API_SERVERS_UP = '(sum(up{job="apiserver"} == 1) / count(up{job="apiserver"})) * 100';
  export const CONTROLLER_MANAGERS_UP =
    '(sum(up{job="kube-controller-manager"} == 1) / count(up{job="kube-controller-manager"})) * 100';
  export const SCHEDULERS_UP = '(sum(up{job="scheduler"} == 1) / count(up{job="scheduler"})) * 100';
  export const API_SERVER_REQUESTS_SUCCESS =
    '(1 - (sum(rate(apiserver_request_total{code=~"5.."}[5m])) or vector(0))/ sum(rate(apiserver_request_total[5m]))) * 100';
  ```

##3 "Operator" depends on ClusterOperator CR resource which can fetch using "oc get clusteroperators", refer the details on concerned codes, [5] and [6].

  [5] console/frontend/packages/operator-lifecycle-manager/src/plugin.tsx
  ```ts
  type: 'Dashboards/Overview/Health/Operator',
  properties: {
    title: 'Operators',                                                   <--- Health card name
    resources: [
      {
        kind: referenceForModel(models.ClusterServiceVersionModel),
        isList: true,
        prop: 'clusterServiceVersions',
      },
      {
        kind: referenceForModel(models.SubscriptionModel),
        prop: 'subscriptions',
        isList: true,
      },
    ],
    getOperatorsWithStatuses: getClusterServiceVersionsWithStatuses,      <--- Evaluable ClusterOperator resource states
    operatorRowLoader: async () =>
      (
        await import(
          './components/dashboard/csv-status' /* webpackChunkName: "csv-dashboard-status" */
        )
      ).default,
  },
  ```

  [6] console/frontend/packages/console-app/src/components/dashboards-page/status.ts
  ```ts
  export const getClusterOperatorHealthStatus: GetOperatorsWithStatuses<ClusterOperator> = (
    resources,
  ) => {
    return (resources.clusterOperators.data as ClusterOperator[]).map((co) =>
      getOperatorsStatus<ClusterOperator>([co], getClusterOperatorStatusPriority),
    );
  };
  ```
