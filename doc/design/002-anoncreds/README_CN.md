# Anoncreds 设计

> 译注 1：前后两天时间 （16hrs） 进行文档翻译。
>  
> 译注 2：Anoncreds 设计中需要注意:  
> 1. 基本概念在 https://github.com/hyperledger/indy-node/blob/master/design/anoncreds.md 有更完整的定义和说明。  
> 1. blob 大块云存储如何理解？  
> 1. tails 的可撤销凭证实现原理是什么？  
> 1. node 的交易中哪些交易与 anoncreds 有关？


你能看到关于 Indy SDK Anoncreds 工作流的需求和设计，包括撤销。

* [Anoncreds 参考资料](#anoncreds-references)
* [设计目标](#design-goals)
* [Anoncreds 工作流](#anoncreds-workflow)
* [API](#api)

## Anoncreds References  
Anoncreds 参考资料

Anoncreds 协议相关链接:

* [Anoncreds Workflow](#anoncreds-workflow)
* [Anoncreds Requirements](https://github.com/hyperledger/indy-node/blob/master/design/anoncreds.md#requirements)
* Indy Node Anoncreds transactions:
  * [SCHEMA](https://github.com/hyperledger/indy-node/blob/master/design/anoncreds.md##schema)
  * [CRED_DEF](https://github.com/hyperledger/indy-node/blob/master/design/anoncreds.md##cred_def)
  * [REVOC_REG_DEF](https://github.com/hyperledger/indy-node/blob/master/design/anoncreds.md##revoc_reg_def)
  * [REVOC_REG_ENTRY](https://github.com/hyperledger/indy-node/blob/master/design/anoncreds.md##revoc_reg_entry)
  * [Timestamp Support in State](https://github.com/hyperledger/indy-node/blob/master/design/anoncreds.md#timestamp-support-in-state)
  * [GET_OBJ](https://github.com/hyperledger/indy-node/blob/master/design/anoncreds.md#get_obj)
  * [Issuer Key Rotation](https://github.com/hyperledger/indy-node/blob/master/design/anoncreds.md#issuer-key-rotation)
* [Anoncreds 数学表达](https://github.com/hyperledger/indy-crypto/blob/master/libindy-crypto/docs/AnonCred.pdf)
* [Anoncreds 协议算法API接口](https://github.com/hyperledger/indy-crypto/blob/master/libindy-crypto/docs/anoncreds-design.md)

## Design Goals  
设计目标

* Indy SDK 和 Indy Node 应使用相同的 Anoncreds 实体的格式（Schema，Credential 定义，撤销注册定义，撤销注册增量）
* Indy SDK 和 Indy Node 应使用相同的实体应用方法。
* 应具备集成额外的凭证签名和不破坏 API 的撤销 schemas 的可能性。
* API 应提供灵活、可插拔方法来操作撤销tails文件。
* API 应提供足以在云平台上计算出撤销见证的值，但同时能避免在边缘设备下载到全部tails文件。

## Anoncreds Workflow  
Anoncreds 工作流

<img src="./anoncreds-workflow.svg">

## API

### Issuer 发行人

```Rust
/// 创建凭证 schema 实体，用来描述凭证列出和满足凭证互操作的属性
///
/// Schema 是一种公开的，通常通过公开的 Indy 账本上的 SCHEMA Tx 来分享给全部 anoncreds 工作流活动者。
///
/// 重要：在账本中上传 Schema 以及之后用正确的 seq_no 来获取 Schema 来保留账本的兼容性。
/// 这样能够呼叫 indy_issuer_create_and_store_credential_def 来建立相应的凭证定义。
///
/// #Params
/// command_handle: command handle to map callback to user context
/// issuer_did: DID of schema issuer
/// name: a name the schema
/// version: a version of the schema
/// attrs: a list of schema attributes descriptions
/// cb: Callback that takes command result as parameter
///
/// #Returns
/// schema_id: identifier of created schema
/// schema_json: schema as json
///
/// #Errors
/// Common*
/// Anoncreds*
#[no_mangle]
pub extern fn indy_issuer_create_schema(command_handle: i32,
                                        issuer_did: *const c_char,
                                        name: *const c_char,
                                        version: *const c_char,
                                        attrs: *const c_char,
                                        cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                             schema_id: *const c_char, schema_json: *const c_char)>) -> ErrorCode
```

```Rust
/// 创建凭证定义的实体，包含了发行者DID、凭证 schema、签署凭证的密语、以及用于撤销凭证的密语。
///
/// 凭证定义的实体包含私钥和公钥。私钥将会存储在钱包里。公钥将会以 Json 方式、通常是通过 CRED_DEF Tx 返回给 Indy 账本，用于分享给所有匿名凭证工作流的参与者.
///
/// 重要：通过正确的 seq_no 从账本获取 Schema 来保护账本的兼容性。
///
/// #Params
/// wallet_handle: wallet handler (created by open_wallet).
/// command_handle: command handle to map callback to user context.
/// issuer_did: a DID of the issuer signing cred_def transaction to the Ledger
/// schema_json: credential schema as a json
/// tag: allows to distinct between credential definitions for the same issuer and schema
/// signature_type: credential definition type (optional, 'CL' by default) that defines credentials signature and revocation math. Supported types are:
/// - 'CL': Camenisch-Lysyanskaya credential signature type
/// config_json: type-specific configuration of credential definition as json:
/// - 'CL':
///   - support_revocation: whether to request non-revocation credential (optional, default false)
/// cb: Callback that takes command result as parameter.
///
/// #Returns
/// cred_def_id: identifier of created credential definition
/// cred_def_json: public part of created credential definition
///
/// #Errors
/// Common*
/// Wallet*
/// Anoncreds*
#[no_mangle]
pub extern fn indy_issuer_create_and_store_credential_def(command_handle: i32,
                                                          wallet_handle: i32,
                                                          issuer_did: *const c_char,
                                                          schema_json: *const c_char,
                                                          tag: *const c_char,
                                                          signature_type: *const c_char,
                                                          config_json: *const c_char,
                                                          cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                                               cred_def_id: *const c_char,
                                                                               cred_def_json: *const c_char)>) -> ErrorCode
```

```Rust
/// 创建一个撤销注册表为给定元组的凭证的定义的实体：
/// - 撤销注册表定义，它封装了凭证定义引用、撤销类型特定配置和用于撤销凭证的密语
/// - 撤销注册表声明存储被撤销实体，以一种不揭露原文的方式。声明可以按撤销或确认操作的顺序表示。
///
/// 撤销注册表定义的实体包含私钥和公钥。私钥保存在钱包。公钥将会以 Json 方式、通常是通过 REVOC_REG_DEF Tx 返回给 Indy 账本，用于分享给所有匿名凭证工作流的参与者.
///
/// 撤销注册表声明被存储在钱包，也可以以 REVOC_REG_ENTRY Tx 的顺序被分享。这个呼叫在钱包里初始化声明，返回给初始化的入口。
///
/// 一些撤销注册类型（举例： CL_ACCUM）可以需要二进制文件的生成，叫做 tail，用来隐藏公开在撤销注册表中或被分散在账本以外的撤销凭证信息（REVOC_REG_DEF Tx将会包含 uri 和 tails 的哈希）。
/// 这个呼叫需要访问预配置的一块存储读写器实例操作，这样能够允许写生成的 tails。
///
/// #Params
/// wallet_handle: wallet handler (created by open_wallet).
/// command_handle: command handle to map callback to user context.
/// issuer_did: a DID of the issuer signing transaction to the Ledger
/// revoc_def_type: revocation registry type (optional, default value depends on credential definition type). Supported types are:
/// - 'CL_ACCUM': Type-3 pairing based accumulator. Default for 'CL' credential definition type
/// tag: allows to distinct between revocation registries for the same issuer and credential definition
/// cred_def_id: id of stored in ledger credential definition
/// config_json: type-specific configuration of revocation registry as json:
/// - 'CL_ACCUM': {
///     "issuance_type": (optional) type of issuance. Currently supported:
///         1) ISSUANCE_BY_DEFAULT: all indices are assumed to be issued and initial accumulator is calculated over all indices;
///            Revocation Registry is updated only during revocation.
///         2) ISSUANCE_ON_DEMAND: nothing is issued initially accumulator is 1 (used by default);
///     "max_cred_num": maximum number of credentials the new registry can process (optional, default 100000)
/// }
/// tails_writer_handle: handle of blob storage to store tails
/// cb: Callback that takes command result as parameter.
///
/// #Returns
/// revoc_reg_id: identifier of created revocation registry definition
/// revoc_reg_def_json: public part of revocation registry definition
/// revoc_reg_entry_json: revocation registry entry that defines initial state of revocation registry
///
/// #Errors
/// Common*
/// Wallet*
/// Anoncreds*
#[no_mangle]
pub extern fn indy_issuer_create_and_store_revoc_reg(command_handle: i32,
                                                     wallet_handle: i32,
                                                     blob_storage_writer_handle: i32,
                                                     cred_def_id:  *const c_char,
                                                     tag: *const c_char,
                                                     revoc_def_type: *const c_char,
                                                     config_json: *const c_char,
                                                     cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                                          revoc_reg_def_id: *const c_char,
                                                                          revoc_reg_def_json: *const c_char,
                                                                          revoc_reg_entry_json: *const c_char)>) -> ErrorCode
```

```Rust
/// 创建凭证书面请求将会被 Prover 用于凭证请求。书面请求包括 nonce 和用于按协议的步骤和完整性认证的密钥正确性证明。
///
/// #Params
/// command_handle: command handle to map callback to user context
/// wallet_handle: wallet handler (created by open_wallet)
/// cred_def_id: id of credential definition stored in the wallet
/// cb: Callback that takes command result as parameter
///
/// #Returns
/// credential offer json:
///     {
///         "schema_id": string,
///         "cred_def_id": string,
///         // Fields below can depend on Cred Def type
///         "nonce": string,
///         "key_correctness_proof" : <key_correctness_proof>
///     }
///
/// #Errors
/// Common*
/// Wallet*
/// Anoncreds*
#[no_mangle]
pub extern fn indy_issuer_create_credential_offer(command_handle: i32,
                                                  wallet_handle: i32,
                                                  cred_def_id: *const c_char,
                                                  cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                                       cred_offer_json: *const c_char)>) -> ErrorCode
```

```Rust
/// 检查已给出的 Cred Offer 的 Cred 请求，给 Cred 请求发行凭证。
///
/// Cred 请求必须匹配 Cred Offer。 在 Cred Offer 和 Cred 请求中指向的凭证定义和撤销注册表定义必须是已创建的，并且已存储在钱包中。
///
/// 这个证书撤销的消息必须以 cred_revoc_id 的位置，作为撤销注册表的一部分存储在钱包中。
///
/// 呼叫这支 API 将以 Json 文件格式，返回撤销注册表的差量，通常是以 REVOC_REG_ENTRY Tx 的形式。
/// 注意：可能会累积差量以减轻账本负担。
///
/// #Params
/// wallet_handle: wallet handler (created by open_wallet).
/// command_handle: command handle to map callback to user context.
/// cred_offer_json: a cred offer created by indy_issuer_create_credential_offer
/// cred_req_json: a credential request created by indy_prover_create_credential_req
/// cred_values_json: a credential containing attribute values for each of requested attribute names.
///     Example:
///     {
///      "attr1" : {"raw": "value1", "encoded": "value1_as_int" },
///      "attr2" : {"raw": "value1", "encoded": "value1_as_int" }
///     }
/// rev_reg_id: id of revocation registry stored in the wallet
/// blob_storage_reader_handle: configuration of blob storage reader handle that will allow to read revocation tails
/// cb: Callback that takes command result as parameter.
///
/// #Returns
/// cred_json: Credential json containing signed credential values
///     {
///         "schema_id": string,
///         "cred_def_id": string,
///         "rev_reg_def_id", Optional<string>,
///         "values": <see cred_values_json above>,
///         // Fields below can depend on Cred Def type
///         "signature": <signature>,
///         "signature_correctness_proof": <signature_correctness_proof>
///     }
/// cred_revoc_id: local id for revocation info (Can be used for revocation of this cred)
/// revoc_reg_delta_json: Revocation registry delta json with a newly issued credential
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_issuer_create_credential(command_handle: i32,
                                            wallet_handle: i32,
                                            cred_offer_json: *const c_char,
                                            cred_req_json: *const c_char,
                                            cred_values_json: *const c_char,
                                            rev_reg_id: *const c_char,
                                            blob_storage_reader_handle: i32,
                                            cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                                 cred_revoc_id: *const c_char,
                                                                 revoc_reg_delta_json: *const c_char,
                                                                 cred_json: *const c_char)>) -> ErrorCode
```

```Rust
/// 通过 cred_revoc_id 撤销一个凭证（通过 indy_issuer_create_credential 返回）。
///
/// 相应的凭证定义和撤销注册表必须已经创建并存储在钱包中。
///
/// 呼叫这支 API 将以 Json 文件格式，返回撤销注册表的差量，通常是以 REVOC_REG_ENTRY Tx 的形式。
/// 注意：可能会累积差量以减轻账本负担。
///
/// #Params
/// command_handle: command handle to map callback to user context.
/// wallet_handle: wallet handler (created by open_wallet).
/// blob_storage_reader_cfg_handle: configuration of blob storage reader handle that will allow to read revocation tails
/// rev_reg_id: id of revocation registry stored in wallet
/// cred_revoc_id: local id for revocation info
/// cb: Callback that takes command result as parameter.
///
/// #Returns
/// revoc_reg_delta_json: Revocation registry delta json with a revoked credential
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_issuer_revoke_cred(command_handle: i32,
                                      wallet_handle: i32,
                                      blob_storage_reader_handle: i32,
                                      cred_revoc_id: *const c_char,
                                      cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                           revoc_reg_delta_json: *const c_char)>) -> ErrorCode
```

```Rust
/// 合成两个撤销注册表的差量（以 indy_issuer_create_credential 或 indy_issuer_revoke_credential 返回) 以累积普通差量。
/// 发送普通差量到账本来减少负担。
///
/// #Params
/// command_handle: command handle to map callback to user context.
/// rev_reg_delta_json: revocation registry delta.
/// other_rev_reg_delta_json: revocation registry delta for which PrevAccum value  is equal to current accum value of rev_reg_delta_json.
/// cb: Callback that takes command result as parameter.
///
/// #Returns
/// merged_rev_reg_delta: Merged revocation registry delta
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_issuer_merge_revocation_registry_deltas(command_handle: i32,
                                                           rev_reg_delta_json: *const c_char,
                                                           other_rev_reg_delta_json: *const c_char,
                                                           cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                                                merged_rev_reg_delta: *const c_char)>) -> ErrorCode
```

### Prover  
验证者

```Rust
/// 根据给定的 id 创建一个主密语，并存储在钱包中。
///
/// #Params
/// wallet_handle: wallet handler (created by open_wallet).
/// command_handle: command handle to map callback to user context.
/// master_secret_id: (optional, if not present random one will be generated) new master id
///
/// #Returns
/// out_master_secret_id: Id of generated master secret
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_prover_create_master_secret(command_handle: i32,
                                               wallet_handle: i32,
                                               master_secret_id: *const c_char,
                                               cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                                    out_master_secret_id: *const c_char)>) -> ErrorCode
```

```Rust
/// 为一个给定的 Cred Offer 创建一个 Cred 请求
///
/// 方法为由名称标识的主密语创建了一个盲的主密语。
/// 由名称标识主密语必须已存储在安全的钱包中（参见 prover_create_master_secret）。
/// 盲的主密语作为 cred 请求的一部分。
///
/// #Params
/// command_handle: command handle to map callback to user context
/// wallet_handle: wallet handler (created by open_wallet)
/// prover_did: a DID of the prover
/// cred_offer_json: credential offer as a json containing information about the issuer and a credential
/// cred_def_json: credential definition json
/// master_secret_id: the id of the master secret stored in the wallet
/// cb: Callback that takes command result as parameter.
///
/// #Returns
/// cred_req_json: Credential request json for creation of credential by Issuer
///     {
///      "prover_did" : string,
///      "cred_def_id" : string,
///         // Fields below can depend on Cred Def type
///      "blinded_ms" : <blinded_master_secret>,
///      "blinded_ms_correctness_proof" : <blinded_ms_correctness_proof>,
///      "nonce": string
///    }
/// cred_req_metadata_json: Credential request metadata json for processing of received form Issuer credential.
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_prover_create_credential_req(command_handle: i32,
                                                wallet_handle: i32,
                                                prover_did: *const c_char,
                                                cred_offer_json: *const c_char,
                                                cred_def_json: *const c_char,
                                                master_secret_id: *const c_char,
                                                cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                                     cred_req_json: *const c_char)>) -> ErrorCode
```

```Rust
/// 为给定 cred 请求检查发行者提供的凭证，通过安全钱包中的主密语升级凭证。
///
/// #Params
/// command_handle: command handle to map callback to user context.
/// wallet_handle: wallet handler (created by open_wallet).
/// cred_id: (optional, default is a random one) identifier by which credential will be stored in the wallet
/// cred_req_metadata_json: a credential request metadata created by indy_prover_create_credential_req
/// cred_json: credential json received from issuer
/// cred_def_json: credential definition json
/// rev_reg_def_json: revocation registry definition json
/// cb: Callback that takes command result as parameter.
///
/// #Returns
/// out_cred_id: identifier by which credential is stored in the wallet
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_prover_store_credential(command_handle: i32,
                                           wallet_handle: i32,
                                           cred_id: *const c_char,
                                           cred_req_metadata_json: *const c_char,
                                           cred_json: *const c_char,
                                           cred_def_json: *const c_char,
                                           rev_reg_def_json: *const c_char,
                                           cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                                out_cred_id: *const c_char)>) -> ErrorCode
```

```Rust
/// 根据过滤器，获取人类可读的凭证。
/// 如果过滤器为 NULL，则返回全部凭证。
/// 可以通过 issuer、credential_def、Schema的组合逻辑（和/或）来过滤凭证。
///
/// #Params
/// wallet_handle: wallet handler (created by open_wallet).
/// filter_json: filter for credentials
///        {
///            "schema_id": string, (Optional)
///            "schema_issuer_did": string, (Optional)
///            "schema_name": string, (Optional)
///            "schema_version": string, (Optional)
///            "issuer_did": string, (Optional)
///            "cred_def_id": string, (Optional)
///        }
/// cb: Callback that takes command result as parameter.
///
/// #Returns
/// credentials json
///     [{
///         "referent": string, // cred_id in the wallet
///         "values": <see cred_values_json above>,
///         "schema_id": string,
///         "cred_def_id": string,
///         "rev_reg_id": Optional<string>,
///         "cred_rev_id": Optional<string>
///     }]
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_prover_get_credentials(command_handle: i32,
                                          wallet_handle: i32,
                                          filter_json: *const c_char,
                                          cb: Option<extern fn(
                                              xcommand_handle: i32, err: ErrorCode,
                                              matched_credentials_json: *const c_char)>) -> ErrorCode
```

```Rust
/// 获取与给定请求匹配的人类可读凭证
///
/// #Params
/// wallet_handle: wallet handler (created by open_wallet).
/// proof_request_json: proof request json
///     {
///         "name": string,
///         "version": string,
///         "nonce": string,
///         "requested_attributes": { // set of requested attributes
///              "<attr_referent>": <attr_info>, // see below
///              ...,
///         },
///         "requested_predicates": { // set of requested predicates
///              "<predicate_referent>": <predicate_info>, // see below
///              ...,
///          },
///         "non_revoked": Optional<<non_revoc_interval>>, // see below,
///                        // If specified prover must proof non-revocation
///                        // for date in this interval for each attribute
///                        // (can be overridden on attribute level)
///     }
/// cb: Callback that takes command result as parameter.
///
/// where
/// attr_referent: Proof-request local identifier of requested attribute
/// attr_info: Describes requested attribute
///     {
///         "name": string, // attribute name, (case insensitive and ignore spaces)
///         "restrictions": Optional<[<attr_filter>]> // see below,
///                         // if specified, credential must satisfy to one of the given restriction.
///         "non_revoked": Optional<<non_revoc_interval>>, // see below,
///                        // If specified prover must proof non-revocation
///                        // for date in this interval this attribute
///                        // (overrides proof level interval)
///     }
/// predicate_referent: Proof-request local identifier of requested attribute predicate
/// predicate_info: Describes requested attribute predicate
///     {
///         "name": attribute name, (case insensitive and ignore spaces)
///         "p_type": predicate type (Currently >= only)
///         "p_value": predicate value
///         "restrictions": Optional<[<attr_filter>]> // see below,
///                         // if specified, credential must satisfy to one of the given restriction.
///         "non_revoked": Optional<<non_revoc_interval>>, // see below,
///                        // If specified prover must proof non-revocation
///                        // for date in this interval this attribute
///                        // (overrides proof level interval)
///     }
/// non_revoc_interval: Defines non-revocation interval
///     {
///         "from": Optional<int>, // timestamp of interval beginning
///         "to": Optional<int>, // timestamp of interval ending
///     }
/// filter: see filter_json above
///
/// #Returns
/// credentials_json: json with credentials for the given pool request.
///     {
///         "requested_attrs": {
///             "<attr_referent>": [{ cred_info: <credential_info>, interval: Optional<non_revoc_interval> }],
///             ...,
///         },
///         "requested_predicates": {
///             "requested_predicates": [{ cred_info: <credential_info>, timestamp: Optional<integer> }, { cred_info: <credential_2_info>, timestamp: Optional<integer> }],
///             "requested_predicate_2_referent": [{ cred_info: <credential_2_info>, timestamp: Optional<integer> }]
///         }
///     }, where credential is
///     {
///         "referent": <string>,
///         "attrs": [{"attr_name" : "attr_raw_value"}],
///         "schema_id": string,
///         "cred_def_id": string,
///         "rev_reg_id": Optional<int>,
///         "cred_rev_id": Optional<int>,
///     }
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_prover_get_credentials_for_proof_req(command_handle: i32,
                                                        wallet_handle: i32,
                                                        proof_request_json: *const c_char,
                                                        cb: Option<extern fn(
                                                            xcommand_handle: i32, err: ErrorCode,
                                                            credentials_json: *const c_char)>) -> ErrorCode
```

```Rust

/// 根据给定的证明请求创建一个证明。
/// 或者是一个与可选的透露的属性相对应的凭证，或者是为由每一个属性提供的自证属性。（参见 indy_prover_get_credentials_for_pool_req）。
/// 一个验证请求可能会从不同的 schema 和 不同的发行人那里请求多个凭证。
/// 必须提供全部的被请求的 schemas, 公钥, 撤销注册表。
/// 验证请求同样包含 nonce。
/// 验证的证明要么包含 proof，要么包含为每个被请求的属性的自证属性。
///
/// #Params
/// wallet_handle: wallet handler (created by open_wallet).
/// command_handle: command handle to map callback to user context.
/// proof_request_json: proof request json
///     {
///         "name": string,
///         "version": string,
///         "nonce": string,
///         "requested_attributes": { // set of requested attributes
///              "<attr_referent>": <attr_info>, // see below
///              ...,
///         },
///         "requested_predicates": { // set of requested predicates
///              "<predicate_referent>": <predicate_info>, // see below
///              ...,
///          },
///         "non_revoked": Optional<<non_revoc_interval>>, // see below,
///                        // If specified prover must proof non-revocation
///                        // for date in this interval for each attribute
///                        // (can be overridden on attribute level)
///     }
/// requested_credentials_json: either a credential or self-attested attribute for each requested attribute
///     {
///         "self_attested_attributes": {
///             "self_attested_attribute_referent": string
///         },
///         "requested_attributes": {
///             "requested_attribute_referent_1": {"cred_id": string, "timestamp": Optional<number>, revealed: <bool> }},
///             "requested_attribute_referent_2": {"cred_id": string, "timestamp": Optional<number>, revealed: <bool> }}
///         },
///         "requested_predicates": {
///             "requested_predicates_referent_1": {"cred_id": string, "timestamp": Optional<number> }},
///         }
///     }
/// master_secret_id: the id of the master secret stored in the wallet
/// schemas_json: all schemas json participating in the proof request
///     {
///         <schema1_id>: <schema1_json>,
///         <schema2_id>: <schema2_json>,
///         <schema3_id>: <schema3_json>,
///     }
/// credential_defs_json: all credential definitions json participating in the proof request
///     {
///         "cred_def1_id": <credential_def1_json>,
///         "cred_def2_id": <credential_def2_json>,
///         "cred_def3_id": <credential_def3_json>,
///     }
/// rev_states_json: all revocation states json participating in the proof request
///     {
///         "rev_reg_def1_id": {
///             "timestamp1": <rev_state1>,
///             "timestamp2": <rev_state2>,
///         },
///         "rev_reg_def2_id": {
///             "timestamp3": <rev_state3>
///         },
///         "rev_reg_def3_id": {
///             "timestamp4": <rev_state4>
///         },
///     }
/// cb: Callback that takes command result as parameter.
///
/// where
/// attr_referent: Proof-request local identifier of requested attribute
/// attr_info: Describes requested attribute
///     {
///         "name": string, // attribute name, (case insensitive and ignore spaces)
///         "restrictions": Optional<[<attr_filter>]> // see above,
//                          // if specified, credential must satisfy to one of the given restriction.
///         "non_revoked": Optional<<non_revoc_interval>>, // see below,
///                        // If specified prover must proof non-revocation
///                        // for date in this interval this attribute
///                        // (overrides proof level interval)
///     }
/// predicate_referent: Proof-request local identifier of requested attribute predicate
/// predicate_info: Describes requested attribute predicate
///     {
///         "name": attribute name, (case insensitive and ignore spaces)
///         "p_type": predicate type (Currently >= only)
///         "p_value": predicate value
///         "restrictions": Optional<[<attr_filter>]> // see above,
///                         // if specified, credential must satisfy to one of the given restriction.
///         "non_revoked": Optional<<non_revoc_interval>>, // see below,
///                        // If specified prover must proof non-revocation
///                        // for date in this interval this attribute
///                        // (overrides proof level interval)
///     }
/// non_revoc_interval: Defines non-revocation interval
///     {
///         "from": Optional<int>, // timestamp of interval beginning
///         "to": Optional<int>, // timestamp of interval ending
///     }
///
/// #Returns
/// Proof json
/// For each requested attribute either a proof (with optionally revealed attribute value) or
/// self-attested attribute value is provided.
/// Each proof is associated with a credential and corresponding schema_id, cred_def_id, rev_reg_id and timestamp.
/// There is also aggregated proof part common for all credential proofs.
///     {
///         "requested": {
///             "revealed_attrs": {
///                 "requested_attr1_id": {sub_proof_index: number, raw: string, encoded: string},
///                 "requested_attr4_id": {sub_proof_index: number: string, encoded: string},
///             },
///             "unrevealed_attrs": {
///                 "requested_attr3_id": {sub_proof_index: number}
///             },
///             "self_attested_attrs": {
///                 "requested_attr2_id": self_attested_value,
///             },
///             "requested_predicates": {
///                 "requested_predicate_1_referent": {sub_proof_index: int},
///                 "requested_predicate_2_referent": {sub_proof_index: int},
///             }
///         }
///         "proof": {
///             "proofs": [ <credential_proof>, <credential_proof>, <credential_proof> ],
///             "aggregated_proof": <aggregated_proof>
///         }
///         "identifiers": [{schema_id, cred_def_id, Optional<rev_reg_id>, Optional<timestamp>}]
///     }
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_prover_create_proof(command_handle: i32,
                                       wallet_handle: i32,
                                       proof_req_json: *const c_char,
                                       requested_credentials_json: *const c_char,
                                       master_secret_id: *const c_char,
                                       schemas_json: *const c_char,
                                       credential_defs_json: *const c_char,
                                       rev_states_json: *const c_char,
                                       cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                            proof_json: *const c_char)>) -> ErrorCode
```

```Rust
/// 验证一个（多凭证的）证明。
/// 需提供全部的 schemas, 公钥和撤销凭证。
///
/// #Params
/// wallet_handle: wallet handler (created by open_wallet).
/// command_handle: command handle to map callback to user context.
/// proof_request_json: proof request json
///     {
///         "name": string,
///         "version": string,
///         "nonce": string,
///         "requested_attributes": { // set of requested attributes
///              "<attr_referent>": <attr_info>, // see below
///              ...,
///         },
///         "requested_predicates": { // set of requested predicates
///              "<predicate_referent>": <predicate_info>, // see below
///              ...,
///          },
///         "non_revoked": Optional<<non_revoc_interval>>, // see below,
///                        // If specified prover must proof non-revocation
///                        // for date in this interval for each attribute
///                        // (can be overridden on attribute level)
///     }
/// proof_json: created for request proof json
///     {
///         "requested": {
///             "revealed_attrs": {
///                 "requested_attr1_id": {sub_proof_index: number, raw: string, encoded: string},
///                 "requested_attr4_id": {sub_proof_index: number: string, encoded: string},
///             },
///             "unrevealed_attrs": {
///                 "requested_attr3_id": {sub_proof_index: number}
///             },
///             "self_attested_attrs": {
///                 "requested_attr2_id": self_attested_value,
///             },
///             "requested_predicates": {
///                 "requested_predicate_1_referent": {sub_proof_index: int},
///                 "requested_predicate_2_referent": {sub_proof_index: int},
///             }
///         }
///         "proof": {
///             "proofs": [ <credential_proof>, <credential_proof>, <credential_proof> ],
///             "aggregated_proof": <aggregated_proof>
///         }
///         "identifiers": [{schema_id, cred_def_id, Optional<rev_reg_id>, Optional<timestamp>}]
///     }
/// schemas_json: all schema jsons participating in the proof
///     {
///         <schema1_id>: <schema1_json>,
///         <schema2_id>: <schema2_json>,
///         <schema3_id>: <schema3_json>,
///     }
/// credential_defs_json: all credential definitions json participating in the proof
///     {
///         "cred_def1_id": <credential_def1_json>,
///         "cred_def2_id": <credential_def2_json>,
///         "cred_def3_id": <credential_def3_json>,
///     }
/// rev_reg_defs_json: all revocation registry definitions json participating in the proof
///     {
///         "rev_reg_def1_id": <rev_reg_def1_json>,
///         "rev_reg_def2_id": <rev_reg_def2_json>,
///         "rev_reg_def3_id": <rev_reg_def3_json>,
///     }
/// rev_regs_json: all revocation registries json participating in the proof
///     {
///         "rev_reg_def1_id": {
///             "timestamp1": <rev_reg1>,
///             "timestamp2": <rev_reg2>,
///         },
///         "rev_reg_def2_id": {
///             "timestamp3": <rev_reg3>
///         },
///         "rev_reg_def3_id": {
///             "timestamp4": <rev_reg4>
///         },
///     }
/// cb: Callback that takes command result as parameter.
///
/// #Returns
/// valid: true - if signature is valid, false - otherwise
///
/// #Errors
/// Annoncreds*
/// Common*
/// Wallet*
#[no_mangle]
pub extern fn indy_verifier_verify_proof(command_handle: i32,
                                         proof_request_json: *const c_char,
                                         proof_json: *const c_char,
                                         schemas_json: *const c_char,
                                         credential_defs_json: *const c_char,
                                         rev_reg_defs_json: *const c_char,
                                         rev_regs_json: *const c_char,
                                         cb: Option<extern fn(xcommand_handle: i32, err: ErrorCode,
                                                              valid: bool)>) -> ErrorCode
```

```Rust
/// 创建在指定时间撤销凭证的状态。
///
/// #Params
/// command_handle: command handle to map callback to user context
/// blob_storage_reader_handle: configuration of blob storage reader handle that will allow to read revocation tails
/// rev_reg_def_json: revocation registry definition json
/// rev_reg_delta_json: revocation registry definition delta json
/// timestamp: time represented as a total number of seconds from Unix Epoch
/// cred_rev_id: user credential revocation id in revocation registry
/// cb: Callback that takes command result as parameter
///
/// #Returns
/// revocation state json:
///     {
///         "rev_reg": <revocation registry>,
///         "witness": <witness>,
///         "timestamp" : integer
///     }
///
/// #Errors
/// Common*
/// Wallet*
/// Anoncreds*
#[no_mangle]
pub extern fn indy_create_revocation_state(command_handle: i32,
                                           blob_storage_reader_handle: i32,
                                           rev_reg_def_json: *const c_char,
                                           rev_reg_delta_json: *const c_char,
                                           timestamp: u64,
                                           cred_rev_id: *const c_char,
                                           cb: Option<extern fn(
                                               xcommand_handle: i32, err: ErrorCode,
                                               rev_state_json: *const c_char
                                           )>) -> ErrorCode
```

```Rust
/// 在指定时间为已存在状态的凭证创建新的撤销状态。
///
/// #Params
/// command_handle: command handle to map callback to user context
/// blob_storage_reader_handle: configuration of blob storage reader handle that will allow to read revocation tails
/// rev_state_json: revocation registry state json
/// rev_reg_def_json: revocation registry definition json
/// rev_reg_delta_json: revocation registry definition delta json
/// timestamp: time represented as a total number of seconds from Unix Epoch
/// cred_rev_id: user credential revocation id in revocation registry
/// cb: Callback that takes command result as parameter
///
/// #Returns
/// revocation state json:
///     {
///         "rev_reg": <revocation registry>,
///         "witness": <witness>,
///         "timestamp" : integer
///     }
///
/// #Errors
/// Common*
/// Wallet*
/// Anoncreds*
#[no_mangle]
pub extern fn indy_update_revocation_state(command_handle: i32,
                                           blob_storage_reader_handle: i32,
                                           rev_state_json: *const c_char,
                                           rev_reg_def_json: *const c_char,
                                           rev_reg_delta_json: *const c_char,
                                           timestamp: u64,
                                           cred_rev_id: *const c_char,
                                           cb: Option<extern fn(
                                               xcommand_handle: i32, err: ErrorCode,
                                               updated_rev_state_json: *const c_char
                                           )>) -> ErrorCode
```


### 非结构化云存储 Blob Storage

> Definition - What does Blob Storage mean?
Blob storage is a feature in Microsoft Azure that lets developers store unstructured data in Microsoft's cloud platform. This data can be accessed from anywhere in the world and can include audio, video and text. Blobs are grouped into "containers" that are tied to user accounts. Blobs can be manipulated with .NET code.


“CL 撤销格式”是被用来隐藏在公开撤销注册表中的被撤销凭证信息的“撤销 Tails”。tails

* 是能被用二进制云存储或文件表示的大数整数的固态的（一旦被生成）数组。
* 可能需要十分大量的数据（每个撤销注册表 1GB）。
* 由发行者创建和分享。
* 可以被验证者和发行者请求（必须能被下载）。
* 只能被缓存和下载一次。
* 一些操作（见证增量更新）需要读取云存储块文件中的一小部分。它可以对云中完成 tail 云存储文件和通过网络请求一小部分更加有效。

最终访问云块数据的方法因应用而异。为了符合这一点，SDK 将会提供：

* 云块数据读取的注册自定义处理程序的 API
* 云块数据写入的注册自定义处理程序的 API
* 云块数据一致性验证的 API
* 默认的处理程序实现，允许从本地文件读取云块数据，并将云块数据写到本地文件中。

用以下方式 Tails 发布和访问工作流可以被集成到 Indy Node：

* 发行者生成 tails 向本地文件写入 tails 云存储（通过默认的操作）。我们的 API 将会提供云储存的哈希并生成基于可配置的 URI 模式的 URI。
* 发行者上传云存储到符合 URI 的 CDN 上。
* 发行者发送 REVOC_REG_DEF Tx 并将 tails URI 和哈希发布出去。
* 验证者发送 GET_REVOC_REG_DEF 请求和接收 URI 和哈希。
* 验证者下载被发布的的 tails 文件并在本地存储。
