# Kubernetes `NetworkPolicy` Testing

## Setup Testing Environment {#setup}

1. Deploy infrastructure.

    ```bash
    for grp in circle-apps.yaml square-apps.yaml; do
        kubectl apply -f apps/$grp;
    done
    ```

1. Open four additional terminal windows or tabs, each will be used to interact with a different Pod.

    | Terminal | Namespace | Deployment |
    |----------|-----------|-----|
    | 1 | `netpol-test-circle` | `circle-app-1` |
    | 2 | `netpol-test-circle` | `circle-app-2` |
    | 3 | `netpol-test-square` | `square-app-1` |
    | 4 | `netpol-test-square` | `square-app-2` |

    In terminal 1, execute the following command:
    ```bash
    kubectl exec -it -n netpol-test-circle deploy/circle-app-1 -- /bin/bash
    ```

    In terminal 2, execute the following command:
    ```bash
    kubectl exec -it -n netpol-test-circle deploy/circle-app-2 -- /bin/bash
    ```

    In terminal 3, execute the following command:
    ```bash
    kubectl exec -it -n netpol-test-square deploy/square-app-1 -- /bin/bash
    ```

    In terminal 4, execute the following command:
    ```bash
    kubectl exec -it -n netpol-test-square deploy/square-app-2 -- /bin/bash
    ```

1. Confirm that all four applications can connect to each other. The result should be a `200` message for each application.

    In each terminal, execute the following command:
    ```bash
    for app in \
        circle-app-1.netpol-test-circle \
        circle-app-2.netpol-test-circle \
        square-app-1.netpol-test-square \
        square-app-2.netpol-test-square
    do
        echo -n "${app}: "
        curl -s -m 3 -o /dev/null -w '%{http_code}\n' http://${app}
    done
    ```

    Example output from all apps:
    ```text
    circle-app-1.netpol-test-circle: 200
    circle-app-2.netpol-test-circle: 200
    square-app-1.netpol-test-square: 200
    square-app-2.netpol-test-square: 200
    ```

## Apply `01-deny-ingress.yaml` Policy

### First, Apply to Circle Namespace

This policy will only allow ingress traffic from an empty set, thus effectively denying all ingress traffic.

1. Apply the policy only to the Circle namespace.

    ```bash
    kubectl apply -f policies/01-deny-ingress.yaml -n netpol-test-circle
    ```

1. Run the test in the last step of the [Setup](#setup) section (you might be able to just use the up arrow and hit Enter in your four terminals). This time, apps inside the Circle namespace will still be able to query all apps in both namespaces because we have not affected egress at all. But apps inside the Square namespace will be unable to query apps inside the Circle namespace. The failure output will appear as `000` instead of `200`.

    Example output from Circle namespace apps:
    ```text
    circle-app-1.netpol-test-circle: 200
    circle-app-2.netpol-test-circle: 200
    square-app-1.netpol-test-square: 200
    square-app-2.netpol-test-square: 200
    ```

    Example output from Square namespace apps:
    ```text
    circle-app-1.netpol-test-circle: 000
    circle-app-2.netpol-test-circle: 000
    square-app-1.netpol-test-square: 200
    square-app-2.netpol-test-square: 200
    ```

### Now, Apply to Square Namespace

1. Apply the policy to the Square namespace.

    ```bash
    kubectl apply -f policies/01-deny-ingress.yaml -n netpol-test-square
    ```

1. Run the test in the last step of the [Setup](#setup) section (you might be able to just use the up arrow and hit Enter in your four terminals). Now apps can only talk to each other if they're in the same namespace (intra-namespace communication). Inter-namespace communication is now denied.

    Example output from Circle namespace apps:
    ```text
    circle-app-1.netpol-test-circle: 200
    circle-app-2.netpol-test-circle: 200
    square-app-1.netpol-test-square: 000
    square-app-2.netpol-test-square: 000
    ```

    Example output from Square namespace apps:
    ```text
    circle-app-1.netpol-test-circle: 000
    circle-app-2.netpol-test-circle: 000
    square-app-1.netpol-test-square: 200
    square-app-2.netpol-test-square: 200
    ```

### Remove Policies from All Namespaces

1. Remove `NetworkPolicy` objects from both Circle and Square namespaces before proceeding to the next testing policy.

    ```bash
    for ns in netpol-test-circle netpol-test-square; do 
        kubectl delete -f policies/01-deny-ingress.yaml -n ${ns}
    done
    ```

1. Run the test in the last step of the [Setup](#setup) section (you might be able to just use the up arrow and hit Enter in your four terminals). Confirm that all apps in all namespaces can query each other again.

## Destroy Testing Environment

1. Run `exit` in all the terminal windows or tabs.

1. Destroy infrastructure.

    ```bash
    for grp in circle-apps.yaml square-apps.yaml; do
        kubectl delete -f apps/$grp;
    done
    ```
