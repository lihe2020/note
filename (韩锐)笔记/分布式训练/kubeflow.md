### kubeflow安装纪要

export KF_NAME=kubeflow-test

export BASE_DIR=/opt

export KF_DIR=${BASE_DIR}/${KF_NAME}

export CONFIG_URI="file:/opt/kubeflow-test/kfctl_k8s_istio.0.7.1.yaml"

cd /opt/kubeflow/manifests/kfdef

kfctl build -V --file="./kfctl_k8s_istio.0.7.1.yaml"

存储，调度，网络，配置

### k8s集群

- 账号：root/bita@123.com

- url：https://172.20.4.168:30001/

- token：

  ```text
  eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tcW1wbWciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZjZjMTBhMDktMzEyMi0xMWVhLTg5MDAtMDAxNTVkNTdmYTFiIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.K3CQfLICM2-3ur2oBh1IcuhSHw3sxpM6oCjiUJnyYOc4fAT98C7r4yE27cyO5iTTckNP-6nKwdpbyBDhjO6r3DWvPXLLSO26gbLcrSYlbmYK-fS3VKUs5pTUjS-TAtUdveTQbl7pv5HKMcSo_UPCUQB53iIfmt6AnLblF75CuN83G2BztZZ9U6t7YIVxEyEU0YgMOEoDItONQc5b3SEz9vSYjcYjds49aj-k55kKmh3AtrxwpSzpWlYI3QRVgs2PNJ9z2Vj9BcABpxuiQpxtZj5yl3LvSIZ_RdTDMLn8GIKrv6MmGa3sw1uq5ND76CDzyTxmTsZ7lPtZ6wnDv3mGGw
  ```

  

