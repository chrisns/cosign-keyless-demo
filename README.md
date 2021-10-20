# Cosign Keyless GitHub Action Demo

[![Proof of concept](https://github.com/chrisns/cosign-keyless-demo/actions/workflows/ci.yml/badge.svg)](https://github.com/chrisns/cosign-keyless-demo/actions/workflows/ci.yml)
[![Security Scanning](https://github.com/chrisns/cosign-keyless-demo/actions/workflows/security.yml/badge.svg)](https://github.com/chrisns/cosign-keyless-demo/actions/workflows/security.yml)

> Proof of concept that uses cosign and GitHub's in built OIDC for actions to sign container images, providing a proof that what is in the registry came from your GitHub action.

[Working example](https://github.com/chrisns/cosign-keyless-demo/actions/workflows/ci.yml)

## Usage

The very short answer is you need to add the `id-token: write` permission and then `cosign sign -oidc-issuer https://token.actions.githubusercontent.com`

To see that in action see the abridged spec:

```yaml
name: Build Push Sign
on: { push: { branches: ['main'] } }

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v1

      - name: Login to GitHub
        uses: docker/login-action@v1.9.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: build+push
        uses: docker/build-push-action@v2.7.0
        with:
          context: .
          push: true
          tags: ghcr.io/chrisns/cosign-keyless-demo:latest

      - uses: sigstore/cosign-installer@main

      - name: Sign the images
        run: |
          cosign sign -oidc-issuer \
            https://token.actions.githubusercontent.com \
            ghcr.io/chrisns/cosign-keyless-demo:latest
        env:
          COSIGN_EXPERIMENTAL: 1

      - name: Verify the pushed tags
        run: cosign verify ghcr.io/chrisns/cosign-keyless-demo:latest
        env:
          COSIGN_EXPERIMENTAL: 1
```

We can now do:

```shell
$ COSIGN_EXPERIMENTAL=1 cosign verify -output text ghcr.io/chrisns/cosign-keyless-demo:latest
```

and get back, note the subject is `"https://github.com/chrisns/cosign-keyless-demo/.github/workflows/ci.yml@refs/heads/main"`

```json
[
  {
    "critical": {
      "identity": {
        "docker-reference": "ghcr.io/chrisns/cosign-keyless-demo"
      },
      "image": {
        "docker-manifest-digest": "sha256:25884cd45107dd8d9b1b1e187c32ac8b97a0277fe506ac83da0cab27cf1d4c0d"
      },
      "type": "cosign container image signature"
    },
    "optional": {
      "Bundle": {
        "SignedEntryTimestamp": "MEQCID9dyEFKuXUNa7Ug5JKohmsOgb8kqtCLjB/KWZtMGcsnAiAEjOrcxs/WUVsEZOvdl0lbK6nGIlYi0Ld2McLq229VFg==",
        "Payload": {
          "body": "eyJhcGlWZXJzaW9uIjoiMC4wLjEiLCJraW5kIjoicmVrb3JkIiwic3BlYyI6eyJkYXRhIjp7Imhhc2giOnsiYWxnb3JpdGhtIjoic2hhMjU2IiwidmFsdWUiOiJkMTExNTRhMDEyOGI0MzNiYTRmZGRkNzQzOTM4M2M0OTI3NmVmNGUxNTNiNjFjNGFiNmYyN2ZjNjQwNDIwMTI1In19LCJzaWduYXR1cmUiOnsiY29udGVudCI6Ik1FWUNJUUQrOU13QXRrRk9lN29mWGZQa0VRQVMvc0J2NHdjdG0xQktmeGRMRTU3amd3SWhBSlRPcHRxc2xIKzNhK0tCVW1ycVpnTnRKVmliRWFnd1dQRjFYTHBiallPeCIsImZvcm1hdCI6Ing1MDkiLCJwdWJsaWNLZXkiOnsiY29udGVudCI6IkxTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENrMUpTVU4yUkVORFFXdEhaMEYzU1VKQlowbFVXRnBWUWtoWlNUaHJUVUZGU0ZKQ2RTOXZaRWwwYTNGVE0ycEJTMEpuWjNGb2EycFBVRkZSUkVGNlFYRUtUVkpWZDBWM1dVUldVVkZMUlhkNGVtRlhaSHBrUnpsNVdsTTFhMXBZV1hoRlZFRlFRbWRPVmtKQlRWUkRTRTV3V2pOT01HSXpTbXhOUWpSWVJGUkplQXBOVkVGNVRVUkZkMDFVVVhoT1JtOVlSRlJKZUUxVVFYbE5SRVYzVFhwUmVFMHhiM2RCUkVKYVRVSk5SMEo1Y1VkVFRUUTVRV2RGUjBORGNVZFRUVFE1Q2tGM1JVaEJNRWxCUWxCbVdUaDRibGhoY1Rob2JWUklPR3RhYkd0T2RHRlZRek40VHpaSllVSnlaeXR2UldKTFJURndWWHBFTWtSUGRHbHpha1ZzU3pnS1dHTkpVSFptUkdJMVRXTjNjU3RXYW5ablVEQldiMlJKZW1reWIwWTVlV3BuWjBaMVRVbEpRbUZxUVU5Q1owNVdTRkU0UWtGbU9FVkNRVTFEUWpSQmR3cEZkMWxFVmxJd2JFSkJkM2REWjFsSlMzZFpRa0pSVlVoQmQwMTNSRUZaUkZaU01GUkJVVWd2UWtGSmQwRkVRV1JDWjA1V1NGRTBSVVpuVVZWTWJqTnFDaXRTZG5ocmJDc3JNRGx6V1d0R2FFbzRRazB6TmxVd2QwaDNXVVJXVWpCcVFrSm5kMFp2UVZWNVRWVmtRVVZIWVVwRGEzbFZVMVJ5UkdFMVN6ZFZiMGNLTUN0M2QyZFpNRWREUTNOSFFWRlZSa0ozUlVKQ1NVZEJUVWcwZDJaQldVbExkMWxDUWxGVlNFMUJTMGRqUjJnd1pFaEJOa3g1T1hkamJXd3lXVmhTYkFwWk1rVjBXVEk1ZFdSSFZuVmtRekF5VFVST2JWcFVaR3hPZVRCM1RVUkJkMHhVU1hsTmFtTjBXVzFaTTA1VE1XMU9SMWt4V2xSbmQxcEVTVFZPVkZGMUNtTXpVblpqYlVadVdsTTFibUl5T1c1aVIxWm9ZMGRzZWt4dFRuWmlVemxxV1ZSTk1sbFVSbXhQVkZsNVRrUkthVTlYV21wWmFrVXdUbWs1YWxsVE5Xb0tZMjVSZDFwUldVUldVakJTUVZGSUwwSkdjM2RYV1ZwWVlVaFNNR05JVFRaTWVUbHVZVmhTYjJSWFNYVlpNamwwVERKT2IyTnRiSHBpYmsxMldUSTVlZ3BoVjJSMVRGZDBiR1ZYZUd4ak0wMTBXa2RXZEdKNU9IVmFNbXd3WVVoV2FVd3paSFpqYlhSdFlrYzVNMk41T1dwaFV6VTFZbGQ0UVdOdFZtMWplVGx2Q2xwWFJtdGplVGwwV1Zkc2RVMUJiMGREUTNGSFUwMDBPVUpCVFVSQk1tdEJUVWRaUTAxUlEybG5ZMFIwVXpOaE15OW1hVk4xWkRWbk0wRTNLM1ZaTVNzS2VtOVlZM00zTkc4elFVd3djRmxPZVN0cU5FaHNWaTlLTTFCcWMxWjBXRGt4TVRWYWFYRnpRMDFSUTBScEsxaGhObXhIV1ZKNUsxTlJWMjR5UTI1aFl3cDNWM0V3Ykc0NU9HcDRObkp4UWxSVU1VOHdRamxhVTBsMFdXcG9jbVpyYnpSeFNFZDRlWGQ2YUVOelBRb3RMUzB0TFVWT1JDQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENnPT0ifX19fQ==",
          "integratedTime": 1634724863,
          "logIndex": 783661,
          "logID": "c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d"
        }
      },
      "Subject": "https://github.com/chrisns/cosign-keyless-demo/.github/workflows/ci.yml@refs/heads/main"
    }
  },
  {
    "critical": {
      "identity": {
        "docker-reference": "ghcr.io/chrisns/cosign-keyless-demo"
      },
      "image": {
        "docker-manifest-digest": "sha256:25884cd45107dd8d9b1b1e187c32ac8b97a0277fe506ac83da0cab27cf1d4c0d"
      },
      "type": "cosign container image signature"
    },
    "optional": {
      "Bundle": {
        "SignedEntryTimestamp": "MEYCIQDqC+9t68eV92HIpNbGLw/mmb831CwMKlTciw+U5dmU6AIhANB+DCpJSuSg66cz8Y+5YszFiE8ZVCuQDfTqrZ68yrGW",
        "Payload": {
          "body": "eyJhcGlWZXJzaW9uIjoiMC4wLjEiLCJraW5kIjoicmVrb3JkIiwic3BlYyI6eyJkYXRhIjp7Imhhc2giOnsiYWxnb3JpdGhtIjoic2hhMjU2IiwidmFsdWUiOiJkMTExNTRhMDEyOGI0MzNiYTRmZGRkNzQzOTM4M2M0OTI3NmVmNGUxNTNiNjFjNGFiNmYyN2ZjNjQwNDIwMTI1In19LCJzaWduYXR1cmUiOnsiY29udGVudCI6Ik1FVUNJUUNKQ0hibm5hejFZZVR4bXAyQjJrTWVjNmw1ZHAxSHFBeHNyR2RYOVY3VFFnSWdCZWJsNGFpd1V4R3hlZFpNVXRWK3E2ZzVac0t1bC8yZTZsa0JmeVlrWHNzPSIsImZvcm1hdCI6Ing1MDkiLCJwdWJsaWNLZXkiOnsiY29udGVudCI6IkxTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENrMUpTVU4yUkVORFFXdEhaMEYzU1VKQlowbFVXRnBWUWtoWlNUaHJUVUZGU0ZKQ2RTOXZaRWwwYTNGVE0ycEJTMEpuWjNGb2EycFBVRkZSUkVGNlFYRUtUVkpWZDBWM1dVUldVVkZMUlhkNGVtRlhaSHBrUnpsNVdsTTFhMXBZV1hoRlZFRlFRbWRPVmtKQlRWUkRTRTV3V2pOT01HSXpTbXhOUWpSWVJGUkplQXBOVkVGNVRVUkZkMDFVVVhoT1JtOVlSRlJKZUUxVVFYbE5SRVYzVFhwUmVFMHhiM2RCUkVKYVRVSk5SMEo1Y1VkVFRUUTVRV2RGUjBORGNVZFRUVFE1Q2tGM1JVaEJNRWxCUWxCbVdUaDRibGhoY1Rob2JWUklPR3RhYkd0T2RHRlZRek40VHpaSllVSnlaeXR2UldKTFJURndWWHBFTWtSUGRHbHpha1ZzU3pnS1dHTkpVSFptUkdJMVRXTjNjU3RXYW5ablVEQldiMlJKZW1reWIwWTVlV3BuWjBaMVRVbEpRbUZxUVU5Q1owNVdTRkU0UWtGbU9FVkNRVTFEUWpSQmR3cEZkMWxFVmxJd2JFSkJkM2REWjFsSlMzZFpRa0pSVlVoQmQwMTNSRUZaUkZaU01GUkJVVWd2UWtGSmQwRkVRV1JDWjA1V1NGRTBSVVpuVVZWTWJqTnFDaXRTZG5ocmJDc3JNRGx6V1d0R2FFbzRRazB6TmxVd2QwaDNXVVJXVWpCcVFrSm5kMFp2UVZWNVRWVmtRVVZIWVVwRGEzbFZVMVJ5UkdFMVN6ZFZiMGNLTUN0M2QyZFpNRWREUTNOSFFWRlZSa0ozUlVKQ1NVZEJUVWcwZDJaQldVbExkMWxDUWxGVlNFMUJTMGRqUjJnd1pFaEJOa3g1T1hkamJXd3lXVmhTYkFwWk1rVjBXVEk1ZFdSSFZuVmtRekF5VFVST2JWcFVaR3hPZVRCM1RVUkJkMHhVU1hsTmFtTjBXVzFaTTA1VE1XMU9SMWt4V2xSbmQxcEVTVFZPVkZGMUNtTXpVblpqYlVadVdsTTFibUl5T1c1aVIxWm9ZMGRzZWt4dFRuWmlVemxxV1ZSTk1sbFVSbXhQVkZsNVRrUkthVTlYV21wWmFrVXdUbWs1YWxsVE5Xb0tZMjVSZDFwUldVUldVakJTUVZGSUwwSkdjM2RYV1ZwWVlVaFNNR05JVFRaTWVUbHVZVmhTYjJSWFNYVlpNamwwVERKT2IyTnRiSHBpYmsxMldUSTVlZ3BoVjJSMVRGZDBiR1ZYZUd4ak0wMTBXa2RXZEdKNU9IVmFNbXd3WVVoV2FVd3paSFpqYlhSdFlrYzVNMk41T1dwaFV6VTFZbGQ0UVdOdFZtMWplVGx2Q2xwWFJtdGplVGwwV1Zkc2RVMUJiMGREUTNGSFUwMDBPVUpCVFVSQk1tdEJUVWRaUTAxUlEybG5ZMFIwVXpOaE15OW1hVk4xWkRWbk0wRTNLM1ZaTVNzS2VtOVlZM00zTkc4elFVd3djRmxPZVN0cU5FaHNWaTlLTTFCcWMxWjBXRGt4TVRWYWFYRnpRMDFSUTBScEsxaGhObXhIV1ZKNUsxTlJWMjR5UTI1aFl3cDNWM0V3Ykc0NU9HcDRObkp4UWxSVU1VOHdRamxhVTBsMFdXcG9jbVpyYnpSeFNFZDRlWGQ2YUVOelBRb3RMUzB0TFVWT1JDQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENnPT0ifX19fQ==",
          "integratedTime": 1634724867,
          "logIndex": 783662,
          "logID": "c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d"
        }
      },
      "Subject": "https://github.com/chrisns/cosign-keyless-demo/.github/workflows/ci.yml@refs/heads/main"
    }
  }
]
```

And we can inspect the certificate to check it

```Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            5d:95:01:1d:82:3c:90:c0:04:1d:10:6e:fe:87:48:b6:4a:92:de
    Signature Algorithm: ecdsa-with-SHA384
        Issuer: O=sigstore.dev, CN=sigstore
        Validity
            Not Before: Oct 20 10:14:14 2021 GMT
            Not After : Oct 20 10:34:13 2021 GMT
        Subject:
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:f7:d8:f3:19:d7:6a:af:21:99:31:fc:91:99:64:
                    36:d6:94:0b:7c:4e:e8:86:81:ae:0f:a8:11:b2:84:
                    d6:95:33:0f:60:ce:b6:2b:23:12:52:bc:5d:c2:0f:
                    bd:f0:db:e4:c7:30:ab:e5:63:be:03:f4:56:87:48:
                    ce:2d:a8:17:dc
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage:
                Code Signing
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier:
                2E:7D:E3:F9:1B:F1:92:5F:BE:D3:DB:18:90:58:49:F0:13:37:E9:4D
            X509v3 Authority Key Identifier:
                keyid:C8:C5:1D:00:41:9A:24:29:32:51:24:EB:0D:AE:4A:ED:4A:06:D3:EC

            Authority Information Access:
                CA Issuers - URI:http://privateca-content-603fe7e7-0000-2227-bf75-f4f5e80d2954.storage.googleapis.com/ca36a1e96242b9fcb146/ca.crt

            X509v3 Subject Alternative Name: critical
                URI:https://github.com/chrisns/cosign-keyless-demo/.github/workflows/ci.yml@refs/heads/main
    Signature Algorithm: ecdsa-with-SHA384
         30:66:02:31:00:a2:81:c0:ed:4b:76:b7:fd:f8:92:b9:de:60:
         dc:0e:fe:b9:8d:7e:ce:85:dc:b3:be:28:dc:02:f4:a5:83:72:
         fa:3e:07:95:5f:c9:dc:f8:ec:56:d5:fd:d7:5e:59:8a:ab:02:
         31:00:83:8b:e5:da:ea:51:98:47:2f:92:41:69:f6:0a:76:9c:
         c1:6a:b4:96:7f:7c:8f:1e:ab:a8:14:d3:d4:ed:01:f5:94:88:
         b5:88:e1:ad:f9:28:e2:a1:c6:c7:2c:33:84:2b
```

## References

- https://github.com/sigstore/cosign/blob/main/KEYLESS.md
- https://github.com/sigstore/cosign-installer
- https://blog.alexellis.io/deploy-without-credentials-using-oidc-and-github-actions/
