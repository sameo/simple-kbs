# Simple Key Broker Server

`simple-kbs` verifies the launch measurements of SEV(-ES) guests and conditionally provides secrets.

The KBS has three gRPC endpoints, `GetBundle`, `GetSecret`, `GetOnlineSecret`.
These endpoints are described below and defined in the [protobuf](src/grpc/keybroker.proto).

To validate confidential guests, the guest owner must pre-provision the KBS
with policy information. This KBS uses a database to store policies and secrets.
The database is described below.

## API

### GetBundle
The `GetBundle` endpoint takes a policy and certificate chain.
After verifying the certificate chain, the KBS provides a session file and guest owner public key.
This session file defines a secure channel between the KBS and the PSP, which will be used to securely provide secrets.

### GetSecret
The `GetSecret` endpoint takes the launch measurement of the guest, the expected launch parameters, and a secret request.
The expected launch parameters such as policy and firmware version are used to select the appropriate owner-approved launch digest.
The cryptographically verified launch measurement guarantees the correctness of the launch parameters.

The secret request allows the user to request multiple secrets in different formats.
With SEV and SEV-ES, the secret can only be injected once.
Since the secret must be formulated inside the trusted domain (i.e. the KBS) and there is no way to join together multiple secret blobs, any secrets that the KBC needs at runtime must be in the secret blob generated by the KBS.
Thus, requesting multiple secrets at once is essential.

The JSON bundle secret format is compatible with the Attestation Agent's [Offline SEV KBC](https://github.com/confidential-containers/attestation-agent/blob/main/src/kbc_modules/offline_sev_kbc/README.md).
Secrets intended for the `offline_sev_kbc` should be requested with the GUID `e6f5a162-d67f-4750-a67c-5d065f2a9910`.
`simple-kbs` can also be used with the [Online SEV KBC](https://github.com/confidential-containers/attestation-agent/blob/main/src/kbc_modules/online_sev_kbc/README.md), which expects the GUID `1ee27366-0c87-43a6-af48-28543eaf7cb0`.
The secrets are provided in the OVMF secret table format.

### GetOnlineSecret

This endpoint can be used to request secrets at runtime.
To request online secrets the client should first request a `connection` secret at boot.
The connection secret includes a symmetric encryption key and a connection id.
Like any other boot secret, these values are wrapped in a secret blob, which is encrypted and integrity-protected.
A client inside the guest can extract the connection secret and use it to request further secrets at runtime using the `GetOnlineSecret` endpoint.
To request an online secret, the client provides their id and a list of secret request.
This list is the same as what might be used in a `GetSecret` request.
All the secret types that are available at boot can also be requested at runtime.
The policy for online secrets will be evaluated against the guest's original launch information.


## Policies and Secrets

In SEV terms the policy is a group of flags that specifies properties of a confidential guest.
Here the policy also refers more generally to pre-provisioned guidelines that the KBS uses to evaluate the launch measurement of a VM and to determine whether a secret should be injected.
By default the KBS enforces one tenant-wide policy, which is specified in `default_policy.json`.
```
{
  "allowed_digests": [],
  "allowed_policies": [],
  "min_fw_api_major": 0,
  "min_fw_api_minor": 0,
  "allowed_build_ids": []
}
```
Each policy contains five fields.
The guest VM must meet all five requirements for the policy to be validated.
For instance, the guest must boot with a launch digest that is listed in `allowed_digests` or the KBS will not release secrets.
Note that in fields where multiple options can be specified, such as `allowed_digests`, everything will be allowed if no values are specified.
Thus, the default policy above allows everything.
This can be adjusted so that specific launch digests or firmware versions are required.

Policies can also be specified for individual secrets or groups of secrets via the database.
For a given secret request, the guest must satisfy all associated policies or the secret will not be injected.
Adding additional policies for secrets or keysets cannot make the policy computed for a secret request less restrictive.

Currently the secrets themselves are also stored in the database, although HSM integration is planned.

The database can be configured for MySQL according to [db-mysql.sql](./db-mysql.sql).
KBS is connected to database via environment variables.
* `KBS_DB_TYPE`: Set this to the type of database that you are using for simple-kbs.  This can be mysql, postgres, or sqlite3.
* `KBS_DB_HOST`: This should be set to the host running mysql or postgres database engines. Currently, this needs to be set for sqlite but is not used.
* `KBS_DB_USER`: Mysql or postgres user with permissions to insert,update, and select data from the simple-kbs database. Currently, this needs to be set for sqlite but is not used.
* `KBS_DB_PW`: Mysql or postgres password for `KBS_DB_USER`.  Currently, this needs to be set for sqlite but is not used.
* `KBS_DB`: Name of the postgres or mysql database.  For sqlite, this is the path to the sqlite3 database.

This KBS does not calculate the launch digest. The guest owner must calculate the launch digest ahead of time.
The [sev-snp-measure](https://github.com/IBM/sev-snp-measure) tool can be used to calculate the launch digest of an SEV guest. For example:

    $ sev-snp-measure -v --mode=sev --output-format=base64 \
                      --ovmf=OVMF.fd                       \
                      --kernel=vmlinuz                     \
                      --initrd=kata-containers-initrd.img  \
                      --append="console=ttyS0 loglevel=6"
    Calculated SEV guest measurement: XAI+mQvk/x/kCyHprKj3K7zmXmdm+/7SfpG9AUDWIMQ=

This means that the guest firmware code does not need to be uploaded to the KBS and that SEV and SEV-ES launches follow an identical flow.
The downside is that the guest owner might have to generate more firmware digests ahead of time to account for variations in initrd or CPU count (for SEV-ES guests).

Loosely based on [CCv0 SEV GOP script](https://github.com/confidential-containers-demo/scripts/tree/main/guest-owner-proxy).

## Local development

The [`tools/run_local_tests.sh`](tools/run_local_tests.sh) script creates a DB
server container and runs the integration tests against it.
