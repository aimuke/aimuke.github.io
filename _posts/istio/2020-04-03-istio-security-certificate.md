---
title: "Citadel 中证书的流转"
tags: [istio, citadel, security, certificate]
---

## 生成CA

citadel 中有两种方式设置 `istio-ca-secret`：

- 先在k8s中手动创建一个 `istio-ca-secret`, 这种主要用于在同时部署多个 citadel 的时候。避免大家同时创建，出现竞争的情况。
- citadel 自己创建 `istio-ca-secret`，自己生成根证书和私钥。

NewSelfSignedIstioCAOptions() 生成自签发的证书和对应的私钥，并将其作为 `secret` 保存到 `apiserver` 中。 名称为 `istio-ca-secret` 命名空间为 `istio-system`.

然后将自签名的证书和外部导入的证书 `rootCertFile` 一起保存到名称为 `istio-security` 的 `configmap` 中。保存方式为字节拼接，即将 自签名证书的byte 数组和 外部证书的 bytes数组连接到一起。

> 自签名证书及私钥保存在  `istio-ca-secret` 的 secret 中
> configmap 中保存的是自签名证书和外部证书的组合


## 存储到 configmap中

InsertCATLSRootCert() 会将所有的证书信息都存储到 一个名叫 `istio-security` 命名空间为  `istio-system` 的命名空间中。

更新的时候会先查询对应的 configmap 是否存在，如果不存在就新建一个并插入对应的证书数据，否则就更新已经存在的证书数据


## 生成sa的证书

`upsertSecret()` 用于生成 `serviceaccount` 对应的 `secret` 。 

创建前会先查看命名空间下对应的 `secret` 是否已经存在，如果已经存在则不做处理，否则创建新的secret。

secret 过期处理由 `scrtUpdated` 进行处理。 `scrtUpdated` 是 k8s 中一个 list-watch 中注册的方法。当事件触发时，根据条件判断是否需要更新secret。

这里为了避免由于网络导致的失败，设置了一个3次的重试。直到成功的时候才返回。


# 附录


## istio-ca-secret

ca 对应的认证信息：

- `ca-cert.pem` 证书
- `ca-key.pem` 私钥

```sh
$ kubectl get secret istio-ca-secret -n istio-system -o yaml
```

```yaml
apiVersion: v1
data:
  ca-cert.pem: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMzakNDQWNhZ0F3SUJBZ0lSQU9wRnhlRnpLbExKakJoTHg3VmVWS0l3RFFZSktvWklodmNOQVFFTEJRQXcKR0RFV01CUUdBMVVFQ2hNTlkyeDFjM1JsY2k1c2IyTmhiREFlRncweU1EQXpNVGt4TVRVNE5UbGFGdzB6TURBegpNVGN4TVRVNE5UbGFNQmd4RmpBVUJnTlZCQW9URFdOc2RYTjBaWEl1Ykc5allXd3dnZ0VpTUEwR0NTcUdTSWIzCkRRRUJBUVVBQTRJQkR3QXdnZ0VLQW9JQkFRRFFHLzI4ZTZtc1FOaDhOa3o1U3lOUHdwV0svb3pJVGhMdzRxUEoKYUdhN3lBdFllYy9jbm84UXpCZ25VK0RteDY4NFhxdE5jSURTWHJ5bE52Y3llR1JuWjY5Tml2MVh2amRWYW1nOQpVQS9mVS9KMFY5Tmt1OGNBNkduSjBkZnhLSDZzZ2FJUzdoc2pQWTM2YVJHeDhTaTFjY09aYWdoUmJQQW1lTW5tCmd2dGRqRDVjc0lwOFNhTXpsdFJKNHU4SFZla2UrRGQvZ01LMGxTalNybU9LK0hUc0xMZk5Xd0Rpb3ZEUVpNQ0QKU29sa1NpUVlnOTRDQUVpdmwyR1NOZzEwdmZCV3I2MzdpVS85bGZENnk3cXY2c1FpeVJlTGZDOGFvOEtMMzEvZwpkV2RUd2IyR3kwV1FwV2w4a2IzTnI0U0RoYzNGcG5wZ2h2U0d2R3UzOEdOcjExalJBZ01CQUFHakl6QWhNQTRHCkExVWREd0VCL3dRRUF3SUNCREFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUIKQVFBcXUvRnBjK2JoRzREbzJEdlQ3UnBQQjA1NXRGRlRhWmJweWUxQklJd2o5a0M0U2o0bmo5NTB1UjZ3Y0dYQgpJSjJxNkJZQUprRDJFKzFobTMxajVkMUx5dDk3R21hUk16Uzhaam9VOTM5OFBDSk80MWF2NnJoeVZpa1c3cUxBCkVCT09aTjJ6RDUxcTJsZjVTN2lPNHhiM2hvOEtSVy9Hd2Q3R2ZhT1NVN1I1a0VTSWFneVdUOTd6WlRoemh5UTAKSUVqcGF6Sm9jc1dSV2k4aFBremhGZnZ3aGFSTko5Tm42NkdJZ3g2YkRJN1BYdFdtTy9kLzRpbWdCN2hwSi9JawozajJzYnQ3QU40YlVneU56aU1xY053UWh5bGZyRUppS2F6U1BuUVZHdENjRGtXOVV0MlcyeUtaTkJ5K3h5S0JTCi8zYXdIVFA0dzVXWHhzdDFFVzNDU1JPcQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  ca-key.pem: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcFFJQkFBS0NBUUVBMEJ2OXZIdXByRURZZkRaTStVc2pUOEtWaXY2TXlFNFM4T0tqeVdobXU4Z0xXSG5QCjNKNlBFTXdZSjFQZzVzZXZPRjZyVFhDQTBsNjhwVGIzTW5oa1oyZXZUWXI5Vjc0M1ZXcG9QVkFQMzFQeWRGZlQKWkx2SEFPaHB5ZEhYOFNoK3JJR2lFdTRiSXoyTitta1JzZkVvdFhIRG1Xb0lVV3p3Sm5qSjVvTDdYWXcrWExDSwpmRW1qTTViVVNlTHZCMVhwSHZnM2Y0REN0SlVvMHE1aml2aDA3Q3kzelZzQTRxTHcwR1RBZzBxSlpFb2tHSVBlCkFnQklyNWRoa2pZTmRMM3dWcSt0KzRsUC9aWHcrc3U2cityRUlza1hpM3d2R3FQQ2k5OWY0SFZuVThHOWhzdEYKa0tWcGZKRzl6YStFZzRYTnhhWjZZSWIwaHJ4cnQvQmphOWRZMFFJREFRQUJBb0lCQVFDMG9kY1hKbThiYUIxLwprdEkwLzViaXdBNTAyb1R2eDNTQlNQYkk5cWxWREVsc3ZpNUJYQTdwa1h6VmhlU0w2MzZXK3ZUTS9uMlNHMUM2ClJuOUJlMllLcXVCcCtkM3pydEx3ZksrRnFGeGVoOHJHV1FUUFJuMXd1RW82TnIyc1FHM1M1YUg3dEZneHVsZmwKcGhVSjBqeDNZUXRadWNNR2lmdllLTGQyTVBKbE9xZWpxb2xaVnBPdUlDRkFNTWxhWm5FZ1NHSEJWUncxQ2EwbgpEUTRsa09tSUROTGZuUjdxTENNbEtjb082czhrTkVYeU5hbUNnNlI1MXNrZmJ1NGNGN3VXMkZNOWJLL2RwUTRyCkhwNmpERXpRN1VEUUxCVlhVeTdrUDFXN0ZQQ3Ntd05UdnZudGhxSGtBTXZLUkdIaWVteEw5NU9Da1QxeHUrUUgKUFdSTXI1QUJBb0dCQU5iK3h1UVJGSlNSWHVQR2Yrd1BjZTNYT2NONTN3ZkpMVnlvZDRUd0lSZzZnKzdKbW1EaQo3dFNOdllpR2RsTHR2S3FOU0tLRkpPbUxycGtqbWgya1pGcXBaNWFmaHdnWHNCUm10L3JkSnFiZXlYMG1KM1dnCmRwbnNsNGs5YXBTdGFxcFovUkl1bjdpSml4c1BJM3J6dU9kbEJ4R3U0TEo1VVArWnBQZDgzOStSQW9HQkFQZk4KQWluaWQ4UXRGVk5KU1Y3VmVQYnBYNEQrVDJ1UEh2MUpzR2tkYmNCR01vNVBSTnBjOEJYcS85eWdaN1ZKMXVTMgp3c2tJZU1PQ25tZThXNG5Oc2djWDBLMTNhRmdFVlprYm0zUi9VTkNsUUdJQ01VRXhqa2o1T1cya0Y2UWIyd3RuCmFId0VVbGdyL1Iwa0hHaTVHRFMxbThzQ1BTN1UxYklPNmpjZ0pjVkJBb0dBRURQVk0xemlLeXdsZFk4QkZ2NDIKL05DcWhzUEpmaUc0TEhKNXgyZjlab0VLYmxWOUwrNEtSN1NDNHlZWEJycnA3QVNIdzgrNjcycmFkcW9MTkU2dQpUWExVM3JJWkVCQVE4Z2ludHQweHk0T2d0YkRKYW9EMFR6ZFlXRHhycXRiQzRpRzBBOG5GdWJlTDV6Y2wybDlCCndSYUpDTmtnRC9NNm1uaXV5UVA5THpFQ2dZRUE1RXBaZU83cithNnpHOVREcEh1MGduMEVBRm5LSDBSdWYxaloKRGk0UGczam9jSlQwME51WVVBajlDV3c1dnhtMHdXYmlVc1RjUlBwY0p5T3ZqV2dVWUZaL2FLQStZQUEyUCtUZwpOZFpwUko5SmprR0kwUS92anFrVVVEOUJqRzRoUWdOVmpoT0pMVFB4YjF4cVU4eGFVWTBTWjFlN3VCNWFkVDBxClovalU4MEVDZ1lFQWhXQkFlTEdKN2hLVUNJTEVpZkc2Ly9WNjRsNFhneS9OMmhRNW5PSk1qZmNnYzZIYjJqNWcKVWVERkNsM0UyZVE2ayt3WFBKb0xmSWw2VFdmelR4KzEwWGd0ZTNhOXI2SW5NVStlVVJRc1hnN1A4Y1N3aEJYbgpiNlREUEJ5Q1Z6dnV6OWduOTJUY29naWcxTnNRdVRlR2JjMFlWV21PSHJpak5FWHFTdDFnRHZrPQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
  cert-chain.pem: ""
  key.pem: ""
  root-cert.pem: ""
kind: Secret
metadata:
  creationTimestamp: "2020-03-19T11:58:59Z"
  name: istio-ca-secret
  namespace: istio-system
  resourceVersion: "13304678"
  selfLink: /api/v1/namespaces/istio-system/secrets/istio-ca-secret
  uid: 04cf29f1-69d9-11ea-b652-fa163e56fdff
type: istio.io/ca-root
```

## istio-security

configmap 中存储的数据, 这里的证书就是 `istio-ca-secret` 中的证书

```sh
$ kubectl get configmap istio-security -n istio-system -o yaml
```

```yaml
apiVersion: v1
data:
  caTLSRootCert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMzakNDQWNhZ0F3SUJBZ0lSQU9wRnhlRnpLbExKakJoTHg3VmVWS0l3RFFZSktvWklodmNOQVFFTEJRQXcKR0RFV01CUUdBMVVFQ2hNTlkyeDFjM1JsY2k1c2IyTmhiREFlRncweU1EQXpNVGt4TVRVNE5UbGFGdzB6TURBegpNVGN4TVRVNE5UbGFNQmd4RmpBVUJnTlZCQW9URFdOc2RYTjBaWEl1Ykc5allXd3dnZ0VpTUEwR0NTcUdTSWIzCkRRRUJBUVVBQTRJQkR3QXdnZ0VLQW9JQkFRRFFHLzI4ZTZtc1FOaDhOa3o1U3lOUHdwV0svb3pJVGhMdzRxUEoKYUdhN3lBdFllYy9jbm84UXpCZ25VK0RteDY4NFhxdE5jSURTWHJ5bE52Y3llR1JuWjY5Tml2MVh2amRWYW1nOQpVQS9mVS9KMFY5Tmt1OGNBNkduSjBkZnhLSDZzZ2FJUzdoc2pQWTM2YVJHeDhTaTFjY09aYWdoUmJQQW1lTW5tCmd2dGRqRDVjc0lwOFNhTXpsdFJKNHU4SFZla2UrRGQvZ01LMGxTalNybU9LK0hUc0xMZk5Xd0Rpb3ZEUVpNQ0QKU29sa1NpUVlnOTRDQUVpdmwyR1NOZzEwdmZCV3I2MzdpVS85bGZENnk3cXY2c1FpeVJlTGZDOGFvOEtMMzEvZwpkV2RUd2IyR3kwV1FwV2w4a2IzTnI0U0RoYzNGcG5wZ2h2U0d2R3UzOEdOcjExalJBZ01CQUFHakl6QWhNQTRHCkExVWREd0VCL3dRRUF3SUNCREFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUIKQVFBcXUvRnBjK2JoRzREbzJEdlQ3UnBQQjA1NXRGRlRhWmJweWUxQklJd2o5a0M0U2o0bmo5NTB1UjZ3Y0dYQgpJSjJxNkJZQUprRDJFKzFobTMxajVkMUx5dDk3R21hUk16Uzhaam9VOTM5OFBDSk80MWF2NnJoeVZpa1c3cUxBCkVCT09aTjJ6RDUxcTJsZjVTN2lPNHhiM2hvOEtSVy9Hd2Q3R2ZhT1NVN1I1a0VTSWFneVdUOTd6WlRoemh5UTAKSUVqcGF6Sm9jc1dSV2k4aFBremhGZnZ3aGFSTko5Tm42NkdJZ3g2YkRJN1BYdFdtTy9kLzRpbWdCN2hwSi9JawozajJzYnQ3QU40YlVneU56aU1xY053UWh5bGZyRUppS2F6U1BuUVZHdENjRGtXOVV0MlcyeUtaTkJ5K3h5S0JTCi8zYXdIVFA0dzVXWHhzdDFFVzNDU1JPcQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
kind: ConfigMap
metadata:
  creationTimestamp: "2020-03-19T11:59:00Z"
  name: istio-security
  namespace: istio-system
  resourceVersion: "13304679"
  selfLink: /api/v1/namespaces/istio-system/configmaps/istio-security
  uid: 0533a9fc-69d9-11ea-b652-fa163e56fdff

```


## key-and-cert

`serviceaccount` 对应的secret， 其中包含了 `serviceaccount` 对应的证书，私钥和根证书和。

- `cert-chain.pem` 证书
- `key.pem` 私钥
- `root-cert.pem` 根证书, 就是 `istio-ca-secret` 中的证书


```sh
$ kubectl get secret istio.default -n ns1 -o yaml
```

```yaml
apiVersion: v1
data:
  cert-chain.pem: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURHekNDQWdPZ0F3SUJBZ0lRZGYycGRldjg3djFoQXhyclhyZmE4REFOQmdrcWhraUc5dzBCQVFzRkFEQVkKTVJZd0ZBWURWUVFLRXcxamJIVnpkR1Z5TG14dlkyRnNNQjRYRFRJd01ETXhPVEV4TlRrd05Gb1hEVEl3TURZeApOekV4TlRrd05Gb3dBRENDQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFMNk1DTU9VCjJ2TVVsT2ZkNko0a3FJN1A5QjkyQm1Xc1NTaTVmS3dPSU8xMlFJaTFibmpnRXNieHJRaUlRYjYzbTlTM2hmL1EKb2ZML1QxMkljTnZrOEU4MDh4V2U0U1E0aG8rT0pQNnk4T0c5b242Rk1HY1VmMEJUTHQrSHFHV3lhQjBGQ2dwYwo1VmtHa3RLeWhUSHVYU1VUQ3RsKzlDRUZZOFYrZzM5N3VNRDdES1p6MVZRa2pZYThmaVdUS1VZSGgyN0dFN1EyCldqTEFWbHh3cFNpYnYyOURDdVE1OGFHa1UxNFA4ZW1uTy9KVjlHUTY2OWR1SENRa0o1dHVDaENwNGdVeFJSb1UKU0w3aVc1T3ZXOFhZQkFqUXlGV0J1UlFCVVFDMllXYVdDbm13UXhXRDY3Q3h0WXR3c1V2L2UxY3BaR3ZsMy9VUgowRllQa0dDOEtjdnJJSjBDQXdFQUFhTjVNSGN3RGdZRFZSMFBBUUgvQkFRREFnV2dNQjBHQTFVZEpRUVdNQlFHCkNDc0dBUVVGQndNQkJnZ3JCZ0VGQlFjREFqQU1CZ05WSFJNQkFmOEVBakFBTURnR0ExVWRFUUVCL3dRdU1DeUcKS25Od2FXWm1aVG92TDJOc2RYTjBaWEl1Ykc5allXd3Zibk12ZW1WdVlYQXZjMkV2WkdWbVlYVnNkREFOQmdrcQpoa2lHOXcwQkFRc0ZBQU9DQVFFQWUvNnJHTFJ5Sld5QlVzY3JZSUQ0L3UrUXZYTlQ1c3hSNUJobWRDREQ3M0dVCnpXaHl3ZGUyRmwvSXdjY1pWRGRTeWFwdmdCdFd3VERFMS9CNW44LzJsbGlJUTUrNWpIUEpMeFEzZWpOamdLVjIKRUtWVHRSRmRpSjZYemhMaVVwa3Z2YXQ2cGswdE80d05sSnFhTXRwQlRkcHlYTmlCMyttNVc5UnJ5WE5pRWRBKwpyWFhWbHdyMjBNMHVDSHNVY1IvaGRlYys5U0RkTDZYQWl0RXM1dTBDaC9GMUZ4WHdoYzJBem9Vb2pHaFBqa25NCnJkV1hhTFhqclBmMWVxelphUll5YWpiSTJtdVRzak1WMnZKNjduZVBiZk0wRFVmSFVjNEtwM3B1UHpMWW5YT2kKVTNPTFRUMCtBSE1MajZBeU1VZkQ0L0JRU1NBZDlPamYzaTZ5NENTaFhnPT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  key.pem: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBdm93SXc1VGE4eFNVNTkzb25pU29qcy8wSDNZR1pheEpLTGw4ckE0ZzdYWkFpTFZ1CmVPQVN4dkd0Q0loQnZyZWIxTGVGLzlDaDh2OVBYWWh3MitUd1R6VHpGWjdoSkRpR2o0NGsvckx3NGIyaWZvVXcKWnhSL1FGTXUzNGVvWmJKb0hRVUtDbHpsV1FhUzByS0ZNZTVkSlJNSzJYNzBJUVZqeFg2RGYzdTR3UHNNcG5QVgpWQ1NOaHJ4K0paTXBSZ2VIYnNZVHREWmFNc0JXWEhDbEtKdS9iME1LNURueG9hUlRYZy94NmFjNzhsWDBaRHJyCjEyNGNKQ1FubTI0S0VLbmlCVEZGR2hSSXZ1SmJrNjlieGRnRUNORElWWUc1RkFGUkFMWmhacFlLZWJCREZZUHIKc0xHMWkzQ3hTLzk3Vnlsa2ErWGY5UkhRVmcrUVlMd3B5K3NnblFJREFRQUJBb0lCQUhYWHZ2Zk9VSmF5N09CMQpRZzdEMXliemZ5UVI1eVRzSnhhem1HSUVIdU1kRmc0MlBzc3NzUkF1bVBmRTVQd2hLNU9qcUpDc0kreFhiMnNHCkhkNHd1Vm9UQWg4bDhsRm5UL2pxVFFEa0E4dG9iMTFWMjdoMFdicWJkMHF3NkRsMDI2VE80QVhHcStTaUJ4MmQKWUhpZjFTVS9vSjhnUDdWSVV3cnFFa00rYmVXU2loeCtGUWduYUpUWEJHR0t5WVF2Rkk3OHR0U2lJUGdWS3phLwpDVi9EcWdLUDZsZ2F0aVlBQzNueWVrTGovVnJDOEtjaEF1endjQi82MDJHdjZJZXF0eWNvdGd1aCtlZTFCOFZGCnRNbEtaSDc3cCttb2F6cnVtb0J4MnFuNTVFSnlxa0t2aVRFaVc2TkF3aWdzZVRwN2NWR0tjcVpUOVhWZEdiVkwKTTdSYU14VUNnWUVBKzZ3dEhWUlVrMDRkZDFKak1QcmEyemN4RHNKVzExdzJFOUZQbklvbGR5RHhtcHIyTTN5VwpNd0wwNUx5RURGM2tXYWdmbGR0VDJUVTFqbzU4RlVrUE9aV2VlNmZiTU4zT29uMU5aeWZkS2Z5WE5kT0xwaHZuCmJqZU9mU3lWcW4yVzdacGtRTUMrUVJjb0h5S2t4TmwxSGZmSmJtcVM4U3dKYlRzS0ZDR05EMWNDZ1lFQXdkTEsKOGJDRVF0U0JvdDVJcHk2S0pBcFp0Sm5sU1Z3V2c3b0ptelk3am1oUTNVdnF3L2hWNmF1eG1oczNpUExjc1ROdgpCS3ZDUWhKYWMwQmdNTXhvdjdTUXlpbi9aNHBnS0hidHArVVN5UFhPMHRyS0hVZjlKcVZ0N2o2OExZM3dmVHVMCjdXZCtURXJTWGl4NGRac0FKd1NteEIwTlFIWmptTkl0ZnZjRHV5c0NnWUJuemFsQitxRnpySGw4Mks5dTZWalIKcUI4RTNtVmhLSGhwamlDUENXL1FoZmNBOUw5dGx3cUFlY3kyZDRiamJ1cWJqRHVTM01ibHhRdVZBL0hyK1psYwovL2hCT29ldXpSM0lhWFErZ3ZPMnVLZEpuVHB4UmZzYnU3QjZzcVA4a1JacVpBN0xvblFXZHMybW9leGlBT3RNCmRBSlNGNFVLRWtiRkZkL2ZVOE5SdXdLQmdRQ2ZuNDg5anFiT054N3dWK296clJOZGJSekZyTHgxTng3ZnExWC8KK3FEL3ZnOWl3UVArRXNZR1pEMG04bVZCSnVuMEVhekxodnk3MTB1Z2dSTDIvVkVER0p6cHNiN0NzZVpSVE9pYQpqZ0J6ZW1TenFEWXQrVHlXR0VXNW9QYnUrV2RtYTZUb2hvUXdKcXFybmlveWlNMk9WTGxXNTZvalBaejJuWm1VClo3QXQ4d0tCZ1FEZ2kyRW9MSW9MQkFLVG92c1RuMTUvWithU05kT3IrQWtpV0E0VWl3TUNsYWl2eGFXaDJpdVcKdFd3allCMGRNNVV4WVM3NWorbTlvZzVHZ3hBYUZNaFJoVFFyVkN3bWV3Lzc2ajJjU2dJTWppOVJ2UGVJYWF4VApsM2VVTEdMWFBPdW9ER0FsdXNBeGJGcEQzQ3crUENVc29CaVdOakhOQmlZZDBzdXlrVSsrbFE9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
  root-cert.pem: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMzakNDQWNhZ0F3SUJBZ0lSQU9wRnhlRnpLbExKakJoTHg3VmVWS0l3RFFZSktvWklodmNOQVFFTEJRQXcKR0RFV01CUUdBMVVFQ2hNTlkyeDFjM1JsY2k1c2IyTmhiREFlRncweU1EQXpNVGt4TVRVNE5UbGFGdzB6TURBegpNVGN4TVRVNE5UbGFNQmd4RmpBVUJnTlZCQW9URFdOc2RYTjBaWEl1Ykc5allXd3dnZ0VpTUEwR0NTcUdTSWIzCkRRRUJBUVVBQTRJQkR3QXdnZ0VLQW9JQkFRRFFHLzI4ZTZtc1FOaDhOa3o1U3lOUHdwV0svb3pJVGhMdzRxUEoKYUdhN3lBdFllYy9jbm84UXpCZ25VK0RteDY4NFhxdE5jSURTWHJ5bE52Y3llR1JuWjY5Tml2MVh2amRWYW1nOQpVQS9mVS9KMFY5Tmt1OGNBNkduSjBkZnhLSDZzZ2FJUzdoc2pQWTM2YVJHeDhTaTFjY09aYWdoUmJQQW1lTW5tCmd2dGRqRDVjc0lwOFNhTXpsdFJKNHU4SFZla2UrRGQvZ01LMGxTalNybU9LK0hUc0xMZk5Xd0Rpb3ZEUVpNQ0QKU29sa1NpUVlnOTRDQUVpdmwyR1NOZzEwdmZCV3I2MzdpVS85bGZENnk3cXY2c1FpeVJlTGZDOGFvOEtMMzEvZwpkV2RUd2IyR3kwV1FwV2w4a2IzTnI0U0RoYzNGcG5wZ2h2U0d2R3UzOEdOcjExalJBZ01CQUFHakl6QWhNQTRHCkExVWREd0VCL3dRRUF3SUNCREFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUIKQVFBcXUvRnBjK2JoRzREbzJEdlQ3UnBQQjA1NXRGRlRhWmJweWUxQklJd2o5a0M0U2o0bmo5NTB1UjZ3Y0dYQgpJSjJxNkJZQUprRDJFKzFobTMxajVkMUx5dDk3R21hUk16Uzhaam9VOTM5OFBDSk80MWF2NnJoeVZpa1c3cUxBCkVCT09aTjJ6RDUxcTJsZjVTN2lPNHhiM2hvOEtSVy9Hd2Q3R2ZhT1NVN1I1a0VTSWFneVdUOTd6WlRoemh5UTAKSUVqcGF6Sm9jc1dSV2k4aFBremhGZnZ3aGFSTko5Tm42NkdJZ3g2YkRJN1BYdFdtTy9kLzRpbWdCN2hwSi9JawozajJzYnQ3QU40YlVneU56aU1xY053UWh5bGZyRUppS2F6U1BuUVZHdENjRGtXOVV0MlcyeUtaTkJ5K3h5S0JTCi8zYXdIVFA0dzVXWHhzdDFFVzNDU1JPcQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
kind: Secret
metadata:
  annotations:
    istio.io/service-account.name: default
  creationTimestamp: "2020-03-19T11:59:04Z"
  name: istio.default
  namespace: ns1
  resourceVersion: "13304687"
  selfLink: /api/v1/namespaces/ns1/secrets/istio.default
  uid: 075b2323-69d9-11ea-b652-fa163e56fdff
type: istio.io/key-and-cert
```
