# Configuring Validator and Transactor Permissions

> **Note**
>
> These instructions have been tested on Ubuntu 18.04 (Bionic) only.

This section describes the validator and transactor permissions in
Hyperledger Sawtooth.

-   [Transactor key permissioning](#transactor-key-permissioning-label)controls
    who (which users and clients) can submit transactions and batches to a
    validator. Sawtooth provides both on-chain (network-wide) and off-chain
    (local) settings for transactor key permissioning.
-   [Validator key
    permissioning]({% link docs/1.2/architecture/permissioning_requirement.md%})
    controls which nodes can connect to the Sawtooth network. These settings are
    configured with on-chain settings.

The [Identity]({% link docs/1.2/transaction_family_specifications/identity_transaction_family.md%})
and [Settings]({% link docs/1.2/transaction_family_specifications/settings_transaction_family.md%})
transaction processors (or equivalent) are required to process the on-chain
settings. Note that Settings includes several settings to control who can change
on-chain settings and how many votes are required for a settings change.

> **Note**
>
> Sawtooth also includes another type of transaction control that limits
> which transaction processors can submit transactions. For more
> information, see [Setting the Allowed Transaction Types
> (Optional)]({% link docs/1.2/sysadmin_guide/setting_allowed_txns.md%})".

# Transactor Key Permissioning {#transactor-key-permissioning-label}

<!--
  Licensed under Creative Commons Attribution 4.0 International License
  https://creativecommons.org/licenses/by/4.0/
-->

A running protected network needs a mechanism for limiting which
transactors are allowed to submit batches and transactions to the
validators. There are two different methods for defining the transactors
that a validator will accept.

The first method is configuring a validator to only accept batches and
transactions from predefined transactors that are loaded from a local
validator config file. Once the validator is configured, the list of
allowed transactors is immutable while the validator is running. This
set of permissions are only enforced when receiving a batch from a
connected client, but not when receiving a batch from a peer on the
network.

The second method uses the Identity namespace which handles network-wide
updates and agreement on allowed transactors. The allowed transactors
are checked when a batch is received from a client, received from a
peer, and when a block is validated.

When using both on-chain and off-chain configuration, the validator only
accepts transactions and batches from a client if both configurations
agree it should be accepted. If a batch is received from a peer, only
on-chain configuration is checked.

It is possible that in the time that the batch has been in the pending
queue, the transactor permissions have been updated to no longer allow a
batch signer or a transaction signer that is being included in a block.
For this reason, the allowed transactor roles must also be checked on
block validation.

## Off-chain Transactor Permissioning {#Off-Chain_Transactor_Permissioning}

Validators can be locally configured by creating a validation
configuration file, `validator.toml`, and placing it in the config
directory. Add the configuration for transactor permissioning to this
file using the following format:

``` none
[permissions]

ROLE = POLICY_NAME
```

`ROLE` is a role for transactor permissioning (see [Transactor
Roles](#transactor-roles-label)). `POLICY_NAME`
is the name of policy file in the policy_dir (`/etc/sawtooth/policy` by
default). Multiple roles may be defined in the same format within the
validator configuration file.

The policies are stored in the policy_dir defined by the path config
(see [Configuraing
Sawtooth]({% link docs/1.2/sysadmin_guide/configuring_sawtooth.md%}#path-configuration-file).

Each policy consists of permit and deny rules, similar to policies
defined in the Identity namespace. Each line contains either a
`PERMIT_KEY` or `DENY_KEY` followed with either a public key for an
allowed transactor or the `*` character to specify all possible
transactors. The rules are evaluated in order.

A policy file has the following format:

``` none
PERMIT_KEY {key}
DENY_KEY {key}
```

Specify one key per line. To define multiple permitted or denied keys,
use additional `PERMIT_KEY` or `DENY_KEY` lines.

> **Note**
>
> A policy file implicitly ends with the rule `DENY_KEY *`, which denies
> all transactors or validators that are not explicitly specified in a
> `PERMIT_KEY` rule. For example, if a transactor policy file contains a
> single rule that permits one transactor, it is implicitly denying all
> other transactors.

## On-chain Transactor Permissioning {#config-onchain-txn-perm-label}

The Identity namespace stores roles as key-value pairs, where the key is
a role name and the value is a policy. All roles that limit who is
allowed to sign transactions and batches should start with transactor as
a prefix.

``` none
transactor.SUB_ROLE = POLICY_NAME
```

SUB_ROLEs are more specific signing roles, for example who is allowed to
sign transactions. Each role equals a policy name that corresponds to a
policy stored in state. The policy contains a list of public keys that
correspond to the identity signing key of the transactors.

Before you can configure on-chain roles, your public key must be set in
the `sawtooth.identity.allowed_keys` setting. Only those transactors
whose public keys are in that setting are allowed to update roles and
policies.

1.  Make sure that the Identity and Settings transaction processors and
    the REST API are running, as described in [Running Sawtooth as a
    Service]({% link docs/1.2/sysadmin_guide/setting_up_sawtooth_network.md %}#running-sawtooth-as-a-service).

2.  Add your public key to the list of those allowed to change settings.

    In the following command, change `{user}` to specify your own public
    key file.

    ``` none
    $ sudo sawset proposal create --key /etc/sawtooth/keys/validator.priv \
      sawtooth.identity.allowed_keys=$(cat ~/.sawtooth/keys/{user}.pub)
    ```

    > **Important**
    >
    > You must run this command on the same node that created the genesis
    > block. Otherwise, an additional `sawset proposal create` command
    > would be required on the \"genesis node\" to add a second node\'s
    > `validator.priv` key to the `sawtooth.identity.allowed_keys`
    > setting. This guide does not show that step.

3.  Once your public key is stored in the setting, use the command
    `sawtooth identity policy create` to set and update roles and
    policies.

    For example, running the following command will create a policy that
    permits all and is named `policy_1`:

    ``` console
    $ sawtooth identity policy create policy_1 "PERMIT_KEY *"
    ```

    You can enter multiple permit or deny rules in the same command;
    separate each rule with a space.

    > **Important**
    >
    > Be careful not to remove your permission to change Identity
    > settings. Sawtooth will not prevent you from entering a rule to deny
    > all transactors (including yourself).

4.  To see the policy in state, run the following command:

    ``` console
    $ sawtooth identity policy list
    policy_1:
      Entries:
        PERMIT_KEY *
    ```

5.  Use the command `sawtooth identity role create` to create a role for
    this policy. The following example sets the role for `transactor` to
    the policy that permits all:

    ``` console
    $ sawtooth identity role create transactor policy_1
    ```

6.  To see the role in state, run the following command:

    ``` console
    $ sawtooth identity role list
    transactor: policy_1
    ```

## Transactor Roles {#transactor-roles-label}

The following identity roles are used to control which transactors are
allowed to sign transactions and batches on the system.

`default`:

:   When evaluating role permissions, if the role has not been set, the
    default policy is used. The policy can be changed to meet the
    network\'s requirements after initial start-up by submitting a new
    policy with the name default. If the default policy has not been
    explicitly set, the default is `PERMIT_KEY *` (permit all).

`transactor`:

:   The top level role for controlling who can sign transactions and
    batches on the system. This role is used when the allowed
    transactors for transactions and batches are the same. Any
    transactor whose public key is in the policy is allowed to sign
    transactions and batches, unless a more specific sub-role disallows
    their public key.

`transactor.transaction_signer`:

:   If a transaction is received that is signed by a transactor who is
    not permitted by the policy, the batch containing the transaction
    will be dropped.

`transactor.transaction_signer.{tp_name}`:

:   If a transaction is received for a specific transaction family that
    is signed by a transactor who is not permitted by the policy, the
    batch containing the transaction will be dropped. Replace {tp_name}
    with a transaction family name (see [Transaction Family
    Specifications]({% link docs/1.2/transaction_family_specifications/index.md%})).

`transactor.batch_signer`:

:   If a batch is received that is signed by a transactor who is not
    permitted by the policy, that batch will be dropped.

# Validator Key Permissioning

Sawtooth allows the network to limit the nodes that are able to connect
to it. The permissioning rules determine the roles a connection is able
to play on the network. These roles control the types of messages that
can be sent and received over a given connection. In this section, the
entities acting in the different roles are referred to as
\"requesters\".

Validators are able to determine whether messages delivered to them
should be handled or dropped based on a set of role and identities
stored within the Identity namespace. Each requester is identified by
the public key derived from its identity signing key. Permission
verifiers examine incoming messages against the policy and the current
configuration and either permit, drop, or respond with an error. In
certain cases, the connection will be forcibly closed \-- such as if a
node is not allowed to connect to the network.

This on-chain approach allows the whole network to change its policies
at the same time while the network is live, instead of relying on a
startup configuration.

## Configuring Authorization

The Identity namespace stores roles as key-value pairs, where the key is
a role name and the value is a policy. Sawtooth network permissioning
roles use the following pattern:

``` none
network[.SUB_ROLE] = POLICY_NAME
```

where `network` is the name of the role to be used for network-wide
validator permissioning. POLICY_NAME refers to the name of a policy that
is set in the Identity namespace. The policy defines the public keys
that are allowed to participate in that role. The policy is made up of
PERMIT_KEY and DENY_KEY rules and is evaluated in order. If the public
key is denied, the connection will be rejected. For more information,
please look at the [Identity Transaction Family]({% link
docs/1.2/transaction_family_specifications/identity_transaction_family.md %}).

1.  As in the previous procedure (see [On-chain Transactor
    Permissioning](#config-onchain-txn-perm-label), make
    sure that the Identity and Settings transaction processors and the
    REST API are running.

2.  This procedure assumes that you already added your public key to the
    `sawtooth.identity.allowed_keys` setting. If not, add the your
    public key now, using the `sudo sawset proposal create` command in
    [On-chain Transactor Permissioning](#config-onchain-txn-perm-label).

3.  Create a new policy for the network role, as described in [On-chain
    Transactor Permissioning](#config-onchain-txn-perm-label). The
    remaining steps assume that the policy is named `policy_2`.

4.  Run the following command to set the role for network to permit all:

    ``` console
    $ sawtooth identity role create network policy_2
    ```

5.  To see the role in state, run the following command:

    ``` console
    $ sawtooth identity role list
    network: policy_2
    ```

## Network Roles

The following is the suggested high-level role for on-chain network
permissioning.

`default`:

:   When evaluating role permissions, if the role has not been set, the
    default policy is used. The policy can be changed to meet the
    network\'s requirements after initial start-up by submitting a new
    policy with the name default. If the default policy has not been
    explicitly set, the default is `PERMIT_KEY *` (permit all).

`network`:

:   If a validator receives a peer request from a node whose validator
    public key is not permitted by the policy, the message will be
    dropped, an `AuthorizationViolation` will be returned, and the
    connection will be closed.

    This role is checked by the permission verifier when the following
    messages are received:

    -   `GossipMessage`
    -   `GetPeersRequest`
    -   `PeerRegisterRequest`
    -   `PeerUnregisterRequest`
    -   `GossipBlockRequest`
    -   `GossipBlockResponse`
    -   `GossipBatchByBatchIdRequest`
    -   `GossipBatchByTransactionIdRequest`
    -   `GossipBatchResponse`
    -   `GossipPeersRequest`
    -   `GossipPeersResponse`

`network.consensus`:

:   If a validator receives a `GossipMessage` that contains a new block
    published by a node whose validator public key is not permitted by
    the policy, the message will be dropped, an `AuthorizationViolation`
    will be returned, and the connection will be closed.
