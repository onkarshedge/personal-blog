---
title: "Kubernetes Init Container"
date: 2020-12-08T15:24:11+05:30
draft: false
tags: ["Kubernetes"]
---

#### Versions used
> Kubernetes: 1.16.5  

### Sidecar container

It is recommended to run one container per pod. But, sometimes we need more containers in a pod.
The container which runs the application is known as *main container* and the other container is known as a *sidecar* container.  
Sidecar pattern is used to extend the functionality of the existing application without touching it. 
Use cases for sidecar containers are log forwarding, checking TLS certificates etc.

### Init container
We can also have a container known as `init-container` to run before the containers. Init containers should be short-lived.
After the init-container terminates, the main containers start. 
The yaml spec is similar to a container, they can refer to the config maps, secrets. 

```yaml {linenos=table,hl_lines=["16-31"]}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "hello-metric"
  namespace: my-app-dev
  labels:
    app: "hello-metric"
spec:
  replicas: 2
  template:
    metadata:
      name: "hello-metric"
      labels:
        app: "hello-metric"
    spec:
      initContainers:
          - name: init-container
            image: artifacts.intranet.company.com/kubernetes/vault-secret:1.0
            imagePullPolicy: IfNotPresent
            env:
              - name: NS
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
              - name: VAULT_ADDR
                value: {{ .Values.envSecrets.vaultURL | default "https://vault.prod.util.ak8s.compnay.com/" }}
              {{ - if .Values.envSecrets.secretKeyRef }}
              {{ - $secretName:= .Values.envSecrets.secretKeyRef }}
              - name: SECRET_NAME
                value: {{ $secretName }}
              {{ - end }}
      containers:
        - name: "hello-metric"
          image: "hello-metric:6.0"
          imagePullPolicy: Never
          ports:
            - containerPort: 11000
              name: hello-port
              protocol: TCP

      restartPolicy: Always
  selector:
    matchLabels:
      app: "hello-metric"

```
 
I have two Examples: 
#### 1) Create kubernetes secrets from Hashicorp Vault
The database passwords, API keys were stored in Hashicorp vault. Application needs the updated secrets from Vault as environment variables.
Spring cloud has libraries to directly fetch the secrets from Vault. But, not all frameworks/languages might support that. So the infra team came up with the init containers.  
The init-container would connect to vault using role-id. Fetch the secrets from `/namespace/$secret_name` and create kubernetes secrets.  
`secrets.sh | kubectl apply -f -`  
```shell script 
if [ ! -z $SECRET_NAME ]
then
cat <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: $NS-$SECRET_NAME
data:
EOF
	vault kv get -format=table $SECRET_PATH/$NS/$SECRET_NAME | tail -n +12 | while read k v
    do
    	echo "  "$k: `echo -n $v | base64 | tr -d " \t\n\r"`
    done
fi
```

"EOF" is known as a "Here Tag". Basically \<<EOF tells the shell that we are going to enter a multiline string until the "tag" EOF.
 We can name this tag whatever we want, it's often EOF or STOP.  
 
 The main container would then refer to secrets. As the init-container runs before the main container, we know that the secrets will be present.
 
#### 2) Run flyway database migrations 
We had an ETL pentaho job. The Kubernetes job would run Pentaho Job, transformations. We needed to run some flyway migrations before the job started.
We didn't want to extend the pentaho Dockerfile to install `flyway` and run it before the kettle job.  
Hence, we added an init-container to k8s job.
```yaml
initContainers:
  - name: flyway-migrations
    image: artifacts.intranet.company.com/k8s-project-snapshots/flyway
    imagePullPolicy: Always
    securityContext:
      runAsUser: 101
    command: flyway
    args: '["-url=jdbc:mysql://$(DB_HOST):$(DB_PORT)/$(DB_NAME)?useLegacyDatetimeCode=false&serverTimezone=EST5EDT", "-user=$(DB_USERNAME)", "-password=$(DB_PASSWORD)", "-baselineOnMigrate=true", "migrate"]'
    env:
      {{- if $.Values.envVariables }}
      {{- range $key, $val := $.Values.envVariables }}
      - name: {{ $key }}
        value: {{ $val | quote }}
      {{- end }}
      {{- end }}
      {{- if $.Values.envSecrets }}
      {{- if $.Values.envSecrets.secretKeyRef }}
      {{- $secretKeyObj := printf "%s-%s" $namespace $.Values.envSecrets.secretKeyRef }}
      {{- range $key, $val := $.Values.envSecrets.secretvarMapping }}
      - name: {{ $val.name | default $val.key  }}
        valueFrom:
          secretKeyRef:
            name: {{ $secretKeyObj }}
            key: {{ $val.key }}
      {{- end }}
      {{- end }}
``` 