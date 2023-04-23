# vCluster kube-apiserver Ingress

This follows from my blog post [located here](https://vrelevant.net/vcluster-with-automated-ingress-config/)
1. Deploy NGINX

Refer to the NGINX docs for deployment options. The manifest below deploys to a K8s cluster that has a LoadBalancer service available.

```console
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml
```

2. Enable SSL passthrough so we terminate our TLS connection at the kube-apiserver

```console
kubectl edit deploy -n ingress-nginx ingress-nginx-controller
```

Scroll down until you find the containers args section and add `--enable-ssl-passthrough` to the list of args.

```yaml
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --enable-ssl-passthrough
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-nginx-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
```

3. Update your vCluster Helm values file with ingress config

Remove the following if you followed my previous post:

```yaml
service:
  type: LoadBalancer
```

And add:

```yaml
ingress:
  enabled: true
  pathType: ImplementationSpecific
  apiVersion: networking.k8s.io/v1
  ingressClassName: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

4. Configure a wildcard DNS host record for your ingress controller

Retrieve your ingress server's public IP

```console
kubectl get svc -n ingress-nginx 
```

Create an A record in your DNS domain with value `*` for name and the IP addr from the step above.

5. Deploy a vCluster

```console
export CLUSTER_NAME="cluster-a"
```

```console
kubectl create ns $CLUSTER_NAME
```

_Change `vrelevant.lab` in the commands below to match your domain._

```console
helm install $CLUSTER_NAME loft-sh/vcluster-k8s -n $CLUSTER_NAME -f ./vals.yaml \ 
--set "syncer.extraArgs[0]=--tls-san=$CLUSTER_NAME.vrelevant.lab" \ 
--set "syncer.extraArgs[1]=--out-kube-config-server=https://$CLUSTER_NAME.vrelevant.lab" \ 
--set "ingress.host=$CLUSTER_NAME.vrelevant.lab"
```

6. Retrieve the kubeconfig file and verify it works

_You will likely need to wait one to two minutes before the secret is created in your vCluster namespace. Once it is, run the following command to create local kubeconfig named `kc`_

```console
kubectl get secret vc-$CLUSTER_NAME -n $CLUSTER_NAME --template={{.data.config}} | base64 -d > kc
```

```console
kubectl get ns --kubeconfig=./kc
```

Review the contents of the kc file to verify it is pointing to the ingress for `server`:

```console
cat ./kc
```

```console
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJek1EUXlNekl3TXpNeU9Wb1hEVE16TURReU1ESXdNek15T1Zvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBT3p2CjZRU0JjN3NJbjlPb3NjQnJYeGROM2NzcVBuNHJVQ0lvWVNyQjBhQTJPN09kWk4xUzJEbkNBd3pic1BEVnZuM0QKbXF0b29PYUVZZ0ZSQVRrblN6T2toYUFkWDNTY2dFaE5XREE1SnpqK0dXYnBnd1MxRjNkV2QzV3U1UU5sV3pGdQoxVFYvSS9qbytkcWxQU01hSXFzb2sxOWl4ZXd6NmN1T1UyVDNNTWptYkZTSnl4Z0FJZStaRXNpZnlHMzlpZW9KCkxjaUhEellEUElMZS9DVm5FdmxGZUxEYzVsYmVsVHB3T0JBRzdjWWtlUXVPaTVFV1h6MDhxRVlsbEw0WTBNeC8KUzZLY2FLL0dFamdBMSs2NU51NnNOUGhNaFpGdkx3bzBwSGd5cTJjL1RvZGM3SDltakU0dW9Ea1NUaDVUcXZkZQpkRllJRXNJSVBmVUZwYlFUdE1rQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZLaVFHRTIweU8zTlZuejg5dEU0Z3QrdllPR1hNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBQUJrZWVIWWh1VXFhV2Yya0N6dgpBbXRUUXd6WkxPSC92b3ovQ254eE1XeVRhN1N1OURVOVd2dzBWb1lGV1ZHb1hoUktuOEFNbGZIUkxmR0VZN1lzCmJXQzFudk1GcWsyOUVXL0Ira3BnSHJFTy94MXlyeXo2czZEL2dIY05vSFlPTnU3WnBDN0kyY2Ixb2YrditoMmkKOVpYYnlONlpKMmJPL0o5VldUZDdYdDlBTUtBcEJiWFpCcmxnQW9QVjZ4VGtwTWZKZEMwZDRPbTRHSnV0d05KVwp0TEVDUW5kT1VoWWMvKzQwMmN6LzREL3pML3Z3RDJSU0NhYWhEYWREREU0NlBRZmJTaTEydVJ5WnMwSDJYb2ovClhobXBGY0lsZ2JpOUlsaVAzc0h5N1h0TjhjL0t6amV2dHRWbGxrY0RNUlkxeGg4UTNJSWZqMFFLUm54d2pyek4KS1lzPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://cluster-a.vrelevant.lab
  name: my-vcluster
```

That's it! If you combine this with the previous post, you can deploy secure vClusters in a near fully automated manner. The one manual task remaining is updating the kubeconfig file to remove the cert/key of the initial cluster admin and adding the OIDC config. It would be easy enough to write a simple job that runs in our host cluster, detects those kubeconfigs when they're created and modifies them for us. That said, I did open a feature request issue in the vCluster repo to addt this to the helm chart capability.
