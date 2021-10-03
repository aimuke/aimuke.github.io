---
title: "istio 安全认证原理"
tags: [istio, security, citadel, serviceaccount, secret, spiffe, envoy]
---

# serviceaccount

在Kubernetes上运行时，Istio Auth 使用Kubernetes `serviceaccount` 来识别谁在运行服务：

Istio中的 `service account` 格式为`spiffe://<domain>/ns/<namespace>/sa/<serviceaccount>`

- `domain` 目前是 `cluster.local`. 通过 `trust-domain`可以定制。
- `namespace` 是Kubernetes service account的 `namespace` .
- `serviceaccount` 是Kubernetes service account name

# citadel

在于运行在Kubernetes pod上的服务，每个集群的Istio CA（证书颁发机构）自动化执行密钥和证书管理流程。它主要执行四个关键操作：

- 为每个 service account 生成一个 `SPIFFE` 密钥和证书对
- 根据 `serviceaccount` 将密钥和证书对分发给每个pod
- 定期轮换密钥和证书
- 必要时撤销特定的密钥和证书对

# 工作流程

Istio Auth工作流由两个阶段组成，部署和运行时。对于部署阶段，我们分别讨论这两个场景（例如，在Kubernetes中和VM/裸机），因为他们不一样。一旦秘钥和证书部署好，对于两个场景运行时阶段是相同的。在这节中我们简要介绍工作流。

## 部署阶段

### Kubernetes场景

1. Citadel 观察　Kubernetes API Server，为每个现有和新的 `service account` 创建一个 `SPIFFE` 密钥和证书对，并将其发送到API服务器。
2. 当创建pod时，API Server 会根据 `service account` 使用 Kubernetes `secrets` 来挂载密钥和证书对。
3.  Pilot 使用适当的密钥和证书以及安全命名信息生成配置，该信息定义了什么服务帐户可以运行某个服务，并将其传递给 `Envoy` 。

### VM/裸机场景

1. citadel 创建gRPC服务来处理 `CSR` 请求。
2. 节点代理创建私钥和 `CSR` , 发送 `CSR` 到 citadel 用于签名。
3. citadel 验证 `CSR` 中携带的证书，并签名 `CSR` 以生成证书。
4. 节点代理将从 citadel 接收到的证书和私钥发送给 Envoy。
5. 为了轮换，上述 `CSR` 流程定期重复。

## 运行时阶段

1. 来自客户端服务的出站流量被重新路由到它本地的Envoy。
2. 客户端Envoy与服务器端Envoy开始相互TLS握手。在握手期间，它还进行安全的命名检查，以验证服务器证书中显示的服务帐户是否可以运行服务器服务。
3. mTLS连接建立后，流量将转发到服务器端Envoy，然后通过本地TCP连接转发到服务器服务。

# 举例说明

k8s 中每一个命名空间下都有一个默认的 serviceaccount 叫做 `default` 。在没有为pod 指定 `serviceaccount` 的时候。k8s会自动将 pod 的 `serviceaccount` 的值设置为 `default` 。

`kubectl get sa default -n istio-test -o yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2019-11-13T07:18:14Z"
  name: default
  namespace: istio-test
  resourceVersion: "4401"
  selfLink: /api/v1/namespaces/istio-test/serviceaccounts/default
  uid: c1ae63f8-05e5-11ea-be6e-fa163e56fdff
secrets:
- name: default-token-x7jbq
```

## 创建token
citadel 在启动的时候会检测 `apiserver` 中的所有 `serviceaccount` ，并创建一个 `SPIFFE` 的证书对，并用 `istio.` + `serviceaccount` 作为名称创建 `secret` 保存相关信息

`kubectl get secret/istio.default -n istio-test -o yaml`

```yaml
apiVersion: v1
data:
  cert-chain.pem: {base64 encoding of cert-chain.pem data}
  key.pem: {base64 encoding of key.pem data}
  root-cert.pem: {base64 encoding of root-cert.pem data}
kind: Secret
metadata:
  annotations:
    istio.io/service-account.name: default
  creationTimestamp: "2020-03-19T11:59:04Z"
  name: istio.default
  namespace: istio-test
  resourceVersion: "13304687"
  selfLink: /api/v1/namespaces/istio-test/secrets/istio.default
  uid: 075b2323-69d9-11ea-b652-fa163e56fdff
type: istio.io/key-and-cert
```

## 挂载token

在创建POD时，将 citadel 创建的 `secret` 挂载到 pod 中, 这里如果是使用默认 `serviceaccount` ， 则 `secretName` 为 `istio.default`

```yaml
volumeMounts:
- mountPath: /etc/certs/
    name: istio-certs
    readOnly: true

volumes:
- name: istio-certs
  secret:
    optional: true
    secretName: istio.sleep
```

## envoy 中的配置信息

```json
{
    ...
    "metadata": {
        "filter_metadata": {
            "istio": {
                "config": "/apis/networking.istio.io/v1alpha3/namespaces/istio-system/destination-rule/default"
            }
        }
    },
    "name": "outbound|8080||server2-istio-test.service.consul",
    "tls_context": {
        "common_tls_context": {
            "alpn_protocols": [
                "istio"
            ],
            "tls_certificates": [
                {
                    "certificate_chain": {
                        "filename": "/etc/certs/cert-chain.pem"   # 证书
                    },
                    "private_key": {
                        "filename": "/etc/certs/key.pem"   # 私钥
                    }
                }
            ],
            "validation_context": {
                "trusted_ca": {
                    "filename": "/etc/certs/root-cert.pem"   # 根证书，用于做应用证书的认证
                },
                "verify_subject_alt_name": [
                    "spiffe:///ns/default/sa/default"
                ]
            }
        },
        "sni": "outbound_.8080_._.server2-istio-test.service.consul"
    },
    ...
}
```

# envoy 配置说明

## 启用证书验证

除非验证上下文指定了一个或多个可信授权证书，否则上游和下游连接的证书验证都不会启用。

## 配置示例
```yaml
static_resources:
  listeners:
  - name: listener_0
    address: { socket_address: { address: 127.0.0.1, port_value: 10000 } }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        # ...
      tls_context:
        common_tls_context:
          validation_context:
            trusted_ca:
              filename: /usr/local/my-client-ca.crt
  clusters:
  - name: some_service
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    hosts: [{ socket_address: { address: 127.0.0.2, port_value: 1234 }}]
    tls_context:
      common_tls_context:
        validation_context:
          trusted_ca:
            filename: /etc/ssl/certs/ca-certificates.crt
```

## auth.UpstreamTlsContext

[[auth.UpstreamTlsContext proto](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/auth/cert.proto#envoy-api-msg-auth-upstreamtlscontext)]

```json
{
  "common_tls_context": "{...}",
  "sni": "...",
  "allow_renegotiation": "...",
  "max_session_keys": "{...}"
}
```

- `common_tls_context` (auth.CommonTlsContext) Common TLS context settings.
    > Server certificate verification is not enabled by default. Configure `trusted_ca` to enable verification.

- `sni` (string) SNI string to use when creating TLS backend connections.

- `allow_renegotiation` (bool) If true, server-initiated TLS renegotiation will be allowed.
    > TLS renegotiation is considered insecure and shouldn’t be used unless absolutely necessary.

- `max_session_keys` (UInt32Value) Maximum number of session keys (Pre-Shared Keys for TLSv1.3+, Session IDs and Session Tickets for TLSv1.2 and older) to store for the purpose of session resumption. Defaults to 1, setting this to 0 disables session resumption.

> 服务器名称指示（英语：Server Name Indication，缩写：SNI）是TLS的一个扩展协议[1]，在该协议下，在握手过程开始时客户端告诉它正在连接的服务器要连接的主机名称。这允许服务器在相同的IP地址和TCP端口号上呈现多个证书，并且因此允许在相同的IP地址上提供多个安全（HTTPS）网站（或其他任何基于TLS的服务），而不需要所有这些站点使用相同的证书。它与HTTP/1.1基于名称的虚拟主机的概念相同，但是用于HTTPS。所需的主机名未加密，[2]因此窃听者可以查看请求的网站。

## auth.CommonTlsContext
[[auth.CommonTlsContext proto](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/auth/cert.proto#envoy-api-msg-auth-commontlscontext)]

TLS context shared by both client and server TLS contexts.

```json
{
  "tls_params": "{...}",
  "tls_certificates": [],
  "tls_certificate_sds_secret_configs": [],
  "validation_context": "{...}",
  "validation_context_sds_secret_config": "{...}",
  "combined_validation_context": "{...}",
  "alpn_protocols": []
}
```

- `tls_params` (auth.TlsParameters) TLS protocol versions, cipher suites etc.
- `tls_certificates` (auth.TlsCertificate) Multiple TLS certificates can be associated with the same context to allow both RSA and ECDSA certificates.
    > Only a single TLS certificate is supported in client contexts. In server contexts, the first RSA certificate is used for clients that only support RSA and the first ECDSA certificate is used for clients that support ECDSA.
- `tls_certificate_sds_secret_configs` (auth.SdsSecretConfig) Configs for fetching TLS certificates via SDS API.

- `validation_context` (auth.CertificateValidationContext) How to validate peer certificates.
- `validation_context_sds_secret_config` (auth.SdsSecretConfig) Config for fetching validation context via SDS API.
- `combined_validation_context` (auth.CommonTlsContext.CombinedCertificateValidationContext) Combined certificate validation context holds a default CertificateValidationContext and SDS config. When SDS server returns dynamic CertificateValidationContext, both dynamic and default CertificateValidationContext are merged into a new CertificateValidationContext for validation. This merge is done by Message::MergeFrom(), so dynamic CertificateValidationContext overwrites singular fields in default CertificateValidationContext, and concatenates repeated fields to default CertificateValidationContext, and logical OR is applied to boolean fields.

    > Only one of validation_context, validation_context_sds_secret_config, combined_validation_context may be set.

- `alpn_protocols` (string) Supplies the list of ALPN protocols that the listener should expose. In practice this is likely to be set to one of two values (see the codec_type parameter in the HTTP connection manager for more information):

    “h2,http/1.1” If the listener is going to support both HTTP/2 and HTTP/1.1.

    “http/1.1” If the listener is only going to support HTTP/1.1.

    There is no default for this parameter. If empty, Envoy will not expose ALPN.

> 应用层协议协商（Application-Layer Protocol Negotiation，简称ALPN）是一个传输层安全协议(TLS) 的扩展, ALPN 使得应用层可以协商在安全连接层之上使用什么协议, 避免了额外的往返通讯, 并且独立于应用层协议。 ALPN 用于 HTTP/2 连接, 和HTTP/1.x 相比, ALPN 的使用增强了网页的压缩率减少了网络延时。 ALPN 和 HTTP/2 协议是伴随着 Google 开发 SPDY 协议出现的。

## auth.TlsCertificate
[auth.TlsCertificate proto](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/auth/cert.proto#envoy-api-msg-auth-tlscertificate)

```json
{
  "certificate_chain": "{...}",
  "private_key": "{...}",
  "private_key_provider": "{...}",
  "password": "{...}"
}
```

- `certificate_chain` (core.DataSource) The TLS certificate chain.

- `private_key` (core.DataSource) The TLS private key.

- `private_key_provider` (auth.PrivateKeyProvider) BoringSSL private key method provider. This is an alternative to private_key field. This can’t be marked as oneof due to API compatibility reasons. Setting both private_key and private_key_provider fields will result in an error.

- `password` (core.DataSource) The password to decrypt the TLS private key. If this field is not set, it is assumed that the TLS private key is not password encrypted.

## auth.CertificateValidationContext

[auth.CertificateValidationContext proto](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/auth/cert.proto#envoy-api-msg-auth-certificatevalidationcontext)

```json
{
  "trusted_ca": "{...}",
  "verify_certificate_spki": [],
  "verify_certificate_hash": [],
  "verify_subject_alt_name": [],
  "match_subject_alt_names": [],
  "crl": "{...}",
  "allow_expired_certificate": "...",
  "trust_chain_verification": "..."
}
```

- `trusted_ca` (core.DataSource) TLS certificate data containing certificate authority certificates to use in verifying a presented peer certificate (e.g. server certificate for clusters or client certificate for listeners). If not specified and a peer certificate is presented it will not be verified. By default, a client certificate is optional, unless one of the additional options (require_client_certificate, verify_certificate_spki, verify_certificate_hash, or match_subject_alt_names) is also specified.

  It can optionally contain certificate revocation lists, in which case Envoy will verify that the presented peer certificate has not been revoked by one of the included CRLs.

  See the TLS overview for a list of common system CA locations.

- `verify_subject_alt_name` (string) An optional list of Subject Alternative Names. If specified, Envoy will verify that the Subject Alternative Name of the presented certificate matches one of the specified values.
    > Subject Alternative Names are easily spoofable and verifying only them is insecure, therefore this option must be used together with trusted_ca.(Subject Alternative Names 很容易伪造，只检查名称是不安全的。因此这个选项必须和 `trust_ca` 一起使用)

# References

- [envoy api](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/auth/cert.proto#envoy-api-msg-auth-upstreamtlscontext)

- [SNI](https://zh.wikipedia.org/wiki/%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%90%8D%E7%A7%B0%E6%8C%87%E7%A4%BA)

- [ALPN](https://zh.wikipedia.org/wiki/%E5%BA%94%E7%94%A8%E5%B1%82%E5%8D%8F%E8%AE%AE%E5%8D%8F%E5%95%86)
