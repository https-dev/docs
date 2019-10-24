Best Practices for ACME Client Operations
=========================================

Considerations for Long-Running TLS
Certificate Management Software
-----------------------------------

**CONTRIBUTORS**<br>
Matthew Holt — Caddy Web Server<br>
Jacob Hoffman-Andrews — Let's Encrypt<br>
Erica Portnoy — Electronic Frontier Foundation<br>
Daniel McCarney — Let's Encrypt<br>
Ryan Hurst — Google<br>


### Introduction

The ACME (Automated Certificate Management Environment) protocol, or RFC 8555, is an IETF standard. It facilitates the automatic issuance and revocation of TLS certificates. Its unsupervised nature reduces errors, increases service uptime and availability, improves security, and lowers costs.

TLS connection errors will occur if required peer certificates are missing or invalid, so the availability of websites and services will increasingly rely upon successful ACME operations. This issue becomes more important as certificates become shorter-lived and deployed at a larger scale.

We therefore recommend these best practices or considerations for developers and operators of web servers or other long-running ACME clients at any scale to help reduce load on ACME CAs, increase site availability, decrease operator errors, and improve end user experience.


### Best Practices

This document is agnostic of any particular CA or software, but is crafted through experience from various industry, research, and engineering perspectives. It treats ACME operations as expensive and seeks to minimize the number and frequency of them, while maximizing security and availability of sites. Software complexity is a tradeoff worth considering in terms of the needs of a particular deployment.


#### Upon obtaining a certificate, immediately write it to persistent storage.

Keeping a new certificate only in memory is at risk of being lost if the process is terminated. Don't rely on process longevity by storing certificates solely in memory. Write certificates to some persistent storage medium (such as the file system) so that it can be reused even after the process goes down.

Before making any ACME requests, ensure the persistent storage is configured properly and is functional. It is not a bad idea to write something to storage, read it, verify the contents are accurate, then delete it, when the server starts up or when the storage configuration is changed. If this check fails, no ACME operations should be attempted. You may also wish to check for sufficient available space.

As a special consideration for running in containers, be sure the storage is actually persistent between container runs; destroying the certificate's storage when the container is destroyed does no good.


#### If a valid certificate already exists in storage, use that one instead of obtaining a new one.

A server needing a certificate should always check persistent storage for a usable certificate before requesting a new one with ACME. This reduces load on the CA and will often prevent crippling rate limit issues. It also makes better use of the certificate's full lifetime.
Ensure persistent storage is properly secured.

Certificates are usually stored along with their private keys, which are secret. Thus, storage must be properly secured from external access. This often entails properly permissioning the file system and running programs or scripts without extra privileges.

#### Renew certificates after ⅔ of usable lifetime.

Renewing a certificate[^1] is the single highest-risk ACME procedure because it implies that the future uptime of one or more sites depends on it succeeding before the certificate expires. Because domain validation requires external resources (namely DNS), certificate issuance can be fickle, especially as more time passes from the initial issuance.

Technically, the initial issuance faces the same technical challenges as a renewal, but initial issuances are supervised more often than renewals (though not always) and failures of initial issuance are usually more noticeable by sysadmins, so they are fixed sooner.

The lifetime of a certificate is defined to be the span between its NotBefore and NotAfter dates. The first renewal attempt should be positioned in that window late enough so that as much of the validity window is utilized as possible, but early enough so that there is sufficient time to be made aware of, diagnose, and fix errors before the certificate expires.

The renewal window is defined as the span of time between the first renewal attempt and the certificate's NotAfter date. A renewal window that is ⅓ of the certificate's lifetime works well in most cases. Expect that alerts may not be replied to on weekends or holidays, and that some problems may take several days to resolve if third parties are involved.

As a special consideration when certificate lifetimes are extremely short, a longer renewal window may be preferable to allow more time for troubleshooting before certificates expire; but this will increase load on the CA. When making certificate lifetimes shorter, ACME CAs should expect that the inverse relationship of the ratio of new orders to certificate lifetime is more than linear (i.e. making lifetimes 50% shorter will likely increase orders by more than 50%).

Examples:

- For a 90 day certificate, attempt renewal 30 days before expiration.
- For a 24 hour certificate, attempt renewal 8-12 hours before expiration.

[^1]: From a protocol perspective, there is no significant difference between initial issuance and renewal.

#### Set up proper logging, log monitoring, and notifications.

Logging can be done a number of ways, and it is usually the primary method for becoming aware of problems in long-running processes. System administrators or site owners should be notified as soon as possible if there are issues validating their domain name(s).

Log entries should be classified by level (e.g. debug, info, warning, error), and administrators should be notified or alerted (email, push, or other reliable messaging mechanisms) if error-level logs are emitted.

Classifying log entries is not always obvious. For example, what should be classified as an error? Usually, error level entries should generate alerts, and warnings out of certain tolerances should generate alerts or produce an error. Thinking about log levels in terms of alerts and review/analysis will help guide log level decisions.

For example, a failure due to badNonce that gets retried shouldn't generate an alert. But if multiple renewal attempts fail, an alert should be generated. Similarly, generating excess ACME traffic for a very short time might be a warning, but generating much more traffic than the average in the last N minutes should be alerted.

Alerts should have minimal false positives. Return structured error values and write structured logs when possible so that notifications can be sent only when there is an error that requires your attention.

#### Emit logs liberally.

Keep detailed logs of all automation procedures for at least 2-3 times the lifetime of the certificates. This makes it possible to observe and analyze behavior before errors started occurring and compare it to the problematic behavior.

Some important things to include in logs are:

- When maintenance is beginning/ending
- Which names are being maintained
- Expiration status of certificates in the renewal window
- Each ACME HTTP request (method, URL, response code, and maybe payload in dev/debugging scenarios)
- When OCSP staples are updated, and for which names
- Authz and Order URLs and associated timestamps
- Full error information including detail message and context
- Any changes to loaded certificates

Logs should be structured so that only necessary and relevant notifications can be sent. If notifications are sent for unactionable errors, they will tend to be ignored. Parsing errors by substring matching should be avoided so as to not overfit the matching logic to one particular software's error messages.

#### Limit the scope of user configuration and choose good defaults for the maintenance interval and renewal window.

There are many parameters involved with long-term certificate automation. While some should be user-configurable, most should not be. Always choose reasonable defaults that are conservative and which encourage good practices and broad interoperability.

The certificate maintenance interval and renewal window in particular are crucial parameters. They are affected by certificate lifetimes, which are usually out of the control of the ACME client, so they should probably be user-configurable. To properly eliminate this configuration surface, the software would need to assess the lifetimes of certificates it is currently managing and determine the ideal maintenance interval and renewal window on-the-fly. Further, certificate lifetimes may shift while the process is running. Although it is not ideal to expose these parameters for configuration due to the high chance of user error, the cost required to compute them dynamically may be even higher.

In general, try to reduce the surface area for user configuration, but not at the unreasonable cost of excess software complexity.

#### Use one name per certificate.

X.509v3 certificates can use extensions, one of which is Subject Alternative Name (SAN). The client provides a list of names to be validated when requesting a certificate, and the names which are validated by the CA are added as SANs.

Although it is possible to map many names to one certificate (M:1), a one-to-one (1:1) mapping is recommended. 1:1 is easy to manage, less error-prone, reduces failures at scale, reduces the impact of revocation, and speeds up certificate maintenance. It also can enhance security since each certificate can use a different private key; thus if one key is compromised, the security of connections to the other sites is not automatically compromised as well. The only notable downside is that it may use more storage space since you will have more certificates.

M:1 might be required if:

- Storage space is extremely limited (unlikely, because storage is cheap).
- The web host / service provider charges for certificates and you need to reduce that cost.
- You need to manage fewer certificates due to CA-enforced rate limits.
- Your server software only allows one certificate per virtual host, and the virtual host serves multiple domains.

Using M:1 has little benefit in automated settings, complicates the addition and removal of names, slows down certificate issuance, and increases the chance of validation errors. There will be greater logical and operational complexity by combining many names onto one certificate. In addition, all those names necessarily share a single private key, so if one key is lost, all those sites need a new certificate with a different key. Multi-SAN certificates also have a larger size, so they slow down TLS handshakes.

If a single web server or cluster of servers is handling all subdomains of a single registered domain, it is often better to use and manage a single wildcard certificate. With Let’s Encrypt, this implies configuring the ACME DNS challenge, which requires control over the DNS for the registered domain. Managing certificates for many subdomains of the same registered domain without control over the DNS is not recommended.


#### Staple OCSP responses to certificates.

OCSP responses should be stapled to certificates by the web server to reduce the burden on OCSP responders and to increase client performance by reducing connection latency. Stapling OCSP to certificates also enhances client privacy because the clients will not need to ask a third-party at connection-time whether to trust a specific certificate.

Some top tips for stapling OCSP properly can be found here: https://gist.github.com/sleevi/5efe9ef98961ecfb4da8

When updating a batch of OCSP statuses, be careful not to slam OCSP responders. How critical this is depends on whether the OCSP responder is behind a CDN, and if so, whether the subscriber sites are frequently visited by many clients or not. Frequently-visited sites will have their OCSP responses cached by the CDN, so throttling here is less critical; for other sites, throttling is encouraged. Spread requests out as evenly as possible.

#### Pay attention to OCSP status.

If a signed OCSP update shows that a managed certificate is Revoked, it is recommended to try replacing the certificate with a new one immediately. However, client software should be careful about being too presumptuous about the reason for the revocation. For example, revocation does not necessarily mean that the key was compromised. Clients should check the revocationReason field of an OCSP response and take the best corrective action. Sometimes, CAs will blacklist names or keys from obtaining new certificates after a revocation, so clients should not be aggressive when replacing revoked certific
ates.

#### On-Demand TLS

[On-Demand TLS](https://caddyserver.com/docs/automatic-https#on-demand) is a term coined by [the Caddy project](https://github.com/caddyserver/caddy) for managing certificates at handshake-time (foreground) rather than in the background. This is useful when managing certificates for a large and varying number of domains that are not known before-hand, without needing to modify the server configuration each time a domain is added or removed.

It works by checking the ServerName (SNI) value sent by the client against a whitelist when a ClientHello is received for a ServerName that the server does not yet have a certificate for. If the ServerName is allowed, the server initiates an ACME order to obtain a certificate for it, and immediately serves it to the client during the same TLS handshake, while also storing it and caching it for future use.

Instead of adjusting the server's configuration each time the whitelist of recognized domains changes, the server is provided a single static configuration so it can "ask" whether it is allowed to manage the certificate for a given name.

On-Demand TLS is only practical if the CA can issue a certificate for a name quickly enough so that the TLS handshake can continue. It is OK to add some latency to a TLS handshake, but it should not be used if a TLS handshake needs to be aborted and the client needs to retry later after the certificate is obtained.

In contrast to processing a huge list of domain names all at once, On-Demand TLS is generally less prone to bulk ACME operations since they are only done "as needed," however, error handling and the potential for a thundering herd must be given special consideration.

As with background management, foreground management should be synchronized to avoid a thundering herd: only one TLS handshake should initiate an ACME transaction; other handshakes with the same ServerName should wait until that transaction completes and then share the resulting certificate.

Additional internal throttling may be required to temper dynamic pressure on the CA. If limits are reached, TLS handshakes may not be able to be completed. If a thundering herd can be predicted, ACME operations should begin early and be spread out, with exponential backoff implemented as part of error handling and retries.

One easy way to ensure throttling is to have a global lock that makes ACME transactions mutually exclusive. Performing only one ACME transaction at a time is a great threat, simple throttle, but is not complete by itself: errors can cause an ACME transaction to abort immediately, so timer-based throttling (e.g. exponential backoff) is still necessary. Also, be aware that solving an ACME challenge can sometimes take an impractical amount of time (hours -- this is not too uncommon with some DNS providers), which can lead to denial-of-service if all other TLS handshakes are blocked on DNS propagation, for example.

It is often important to keep the growth of the in-memory certificate cache under control with On-Demand TLS. When to add certificates to the cache is obvious, but when to evict them is less so. Since On-demand TLS makes the long-term utility/need of a particular certificate implicit instead of explicit, there are at least two possible eviction policies to consider:

1. Renew expiring certificates only in the foreground (during handshakes that use it). Evict in the background after certificate expires. This is like LRU.
2. Renew expiring certificates in the background. If renewal fails (repeatedly), evict.

Clients, and the ServerName values they send, are untrusted. Never allow arbitrary clients to initiate ACME operations indiscriminately. Always have some trusted authority (such as a whitelist) filter client requests to determine what is allowed. Failure to do this could lead to unique DoS attacks.


#### Avoid using a production CA endpoint while testing or developing.

Production CA endpoints, especially public endpoints, are often rate-limited. Never use these while testing configuration, experimenting to see if everything is set up correctly, or developing your site/project.

Use a staging or local ACME endpoint instead.

For example, Let's Encrypt has a staging endpoint documented here: https://letsencrypt.org/docs/staging-environment/ - be sure to treat this endpoint with respect, however, as it consumes real resources and still has rate limits.

For extensive testing and development, it is even better to set up a local ACME server before transitioning to a public staging endpoint and then finally a production endpoint.

These projects may be used to stand up a local ACME server:

- https://smallstep.com/blog/private-acme-server/
- https://github.com/letsencrypt/pebble
- https://github.com/letsencrypt/boulder


#### Retry failed ACME operations, but gently.

It is not uncommon for network transactions to intermittently fail. It is worth retrying ACME operations with exponential backoff between attempts and a maximum cap on the interval at about one order of magnitude less than the certificate lifetime. For example, if a certificate lives 90 days, you might retry a failed renewal after one minute, then after 10 minutes, then after 1 hour, then after 1 day, but with no more than 1 day between attempts. If there is no cap on the interval, there may not be enough retris to reliably renew the certificate in time.

A small random variance in the exact spacing of attempts helps avoid thundering herds, and is ideal for relieving load on CAs.

If possible, avoid using `goto` statements or naked `for` loops; they are prone to looping bugs. Instead, us a loop counter like `for i:= 0; i < 3; i++`.

Consider writing errors with their timestamps to persistent storage so that exponential backoff can be resumed after process restarts. This is especially important in accidental tight restart loops.


#### Certificate management features should be embedded within the application using the certificates.

Because certificate maintenance is a long-running process, it is best for its functionality to live close to the application, rather than being more exposed to a countless number of external changes to the system or environment.

Experience has shown that separate processes, scripts, and other "moving parts" considerably raises error surface. It is recommended that certificate management features be made available as dedicated, specialized libraries which are directly embeddable into the applications which need it, rather than trying to hack various scripts, utilities, and dependencies together to form a working whole.

However, if there is an external ACME tool or program (such as an ACME-enabled reverse proxy) that is vastly more capable, reliable, or robust than available embeddable libraries, it is probably better to use the external tool.

Because ACME already relies on external resources to succeed, the goal here is to reduce that reliance on external systems as much as possible.


#### Schedule background certificate maintenance.

Certificates expire, so they will need to be renewed for continued use. "Renewal" actually means replacing the certificate with a new one that has a later NotAfter date. These renewals should happen in the background well before expiration. There are a few ways to go about this.

The simplest approach is to scan the certificate cache/storage on a regular interval; any certificates that are expiring "soon" enter a renewal window and get renewed immediately, before the next interval. One nice thing about this method is that if a renewal fails (even after retries), it can be skipped and tried again at the next scan without any extra complexity. Scans should be inexpensive, so the interval can be somewhat frequent as long as the number of certificates is tractable.

The maintenance interval must be at least one order of magnitude less than the minimum certificate lifetime. Ideally, there would be at least 10 intervals during any certificate's validity period in case retries are necessary. "Very frequent" intervals are on the order of every few seconds to a minute. With longer certificate lifetimes, it is OK for intervals to be on the order of several hours.

A more complex but flexible approach is to individually schedule each certificate's next maintenance operation based on its status and lifetime. This avoids scanning needlessly at regular intervals since the time for each certificate's maintenance is individually scheduled. It also plays well with certificates of varying lifetimes. However, this method also requires more resources and must be able to gracefully reschedule each operation based on whether it succeeded or failed.

Regardless of the method you choose, it is crucial to inject some randomness into the timing of the operations. If all clients try to renew certificates at a certain time, it could overwhelm the CA. For example, avoid setting cron jobs to run at a fixed time.


#### Do not rely on pre-validation checks.

Pre-validation checks are automated checks that attempt to determine if an ACME challenge will succeed or fail before proceeding with the actual challenge.

In practice, these do not work very well for public domains.

It can be difficult to know whether an ACME challenge will succeed. Successful validations require one or more external lookups/connections on infrastructure that depends on the machine's perspective. For example, DNS lookups from the ACME client often result in different records than what the ACME server will see. External connections are difficult or impossible to reliably test internally.

It is not advisable to use a public CA's staging endpoint as a "pre-check" for all certificate issuances, as this would add considerable load at a global scale. However, it is not a bad idea to stand up one's own external ACME server for pre-checks, with the understanding that its result may not match the real ACME server's validation.

So far, pre-validation checks are often inaccurate and seldom worth the effort; however, it is possible that better techniques may arise in the future.

In the meantime, the best way to know whether a challenge will succeed is simply to try it. Before doing so, where possible, a human administrator should ensure that DNS and firewalls are properly configured. (Production ACME endpoints are not to be used for debugging or troubleshooting.)


#### Use all enabled challenge types.

Being issued a certificate is conditional upon solving one of the ACME challenges. The more challenge types that are enabled, the better. If one challenge fails, another may be tried instead.

It is strongly encouraged to enable both the HTTP and TLS-ALPN challenges by default if the server can accept external connections on those ports. If one challenge fails, it is highly recommended to failover and try the other one before giving up. We have seen real cases in the wild with both large and small deployments where simply trying another challenge type kept sites online where they would have otherwise gone down. This is an easy fix with a large benefit.

The DNS challenge is a special case. It requires explicit configuration and is usually only enabled when it is known that the HTTP and TLS-ALPN challenges will not work (for instance, if external connections are rejected), or if wildcard certificates are required. If the DNS challenge is configured, it is unlikely that using the HTTP or TLS-ALPN challenges will be successful, but this depends on the scenario.

#### Don't use MustStaple by default.

Use of OCSP MustStaple has potentially serious consequences that the system administrator should judge before enabling it. Because clients honoring MustStaple require a valid OCSP staple on a served certificate, failure of the server to properly obtain, store, and update OCSP staples can render sites unavailable. Historically, server implementations of OCSP stapling have been fickle, or at best, easy to misconfigure; and certificate revocation as a whole system has several weaknesses at scale. For these reasons, it is advisable to make MustStaple exclusively opt-in, and only after understanding the implications. Site owners should also consider their individual threat and risk models to determine if MustStaple is a necessary addition to their defense model.


#### Coordinate management in a cluster.

Servers in a cluster using certificates for the same name(s) should share the same certificate and key, and either coordinate their ACME operations or delegate them to a single instance.

Ideally, a certificate should be issued only once, and then shared with other servers that serve the same hostname(s) and have access to the same private key. This reduces the load on CAs and greatly improves the efficiency of ACME-dependent deployments at scale.

There are two main modes for implementing this:

1. One server is designated as the ACME client. It is this server's job to obtain & renew the certificate and distribute it and its key to all servers which use it. This is simple but can have uptime issues if the one delegated instance goes down.

2. All servers act as ACME clients and coordinate their ACME operations with the other instances, ensuring that only one instance is responsible for a particular certificate at any given time. If one server begin to renew a certificate, other servers which also need to renew the certificate wait until the first instance completes its operation. The first instance then makes the new certificate available to the others in the fleet. This may be more complex depending on how it is implemented, but can also have greater reliability as there is no one instance responsible for all certificates.


#### Define behavior for interactive and non-interactive modes.

ACME clients operate in a variety of settings, including short-lived, interactive CLI tools and long-lived, non-interactive server daemons. However, some CLI tools are automated in non-interactive scripts, and some server daemons are interactive at startup. These factors may drastically affect behavior related to error handling, concurrency, and throttling.

The ACME client should provide a good user experience, maximize availability, and minimize potential for user and automation errors. This is easiest when framing these issues in terms of interactive and non-interactive modes.

In interactive modes (i.e. when a user is there operating it), it might be best for the program to terminate on the first error so the user can fix it and then continue. For servers that are just starting up, this might also entail blocking (not serving sites) until all initial certificate operations have successfully completed. Printing an error as soon as one happens and then quitting can be nice and user-friendly in interactive modes.

However, in non-interactive mode (i.e. when there is no user actively overseeing the process), aborting on first error can be disastrous in terms of uptime, rate limits, and troubleshooting. In these cases, a server should do its best to start up and serve what it can, while retrying in the background under appropriate throttles and within reasonable limits, and logging all errors and attempts in as much detail as is needed for debugging. This may result in serving sites with an expired or self-signed certificate, or without a certificate at all; however, these symptoms can make troubleshooting easier.

As an example, a server may be started for 3 small sites in an interactive mode because the system administrator is manually deploying this simple setup. As the server starts, it fails to obtain a certificate for site #2, prints an error, and exits. The user fixes the error, restarts the server, and when the server finishes with all 3 certificates, it starts serving sites. The user can then leave, but will monitor logs for any errors in the future. This mode treats certificate errors like socket bind errors. If a server cannot bind a socket, it cannot serve TCP connections; if a server does not have certificates, it cannot serve TLS connections. This reasoning works well in interactive mode, but because of the dynamic nature of the Internet and the external resources required to obtain certificates, it does not work well in non-interactive mode.

As an example for non-interactive mode, an advanced deployment may require 10,000 certificates to satisfy a large, dynamically-generated config that changes every few minutes. The server software is supervised with an init system like systemd, which restarts the server if it quits or if the machine is rebooted (common for security updates). When the server is started, it immediately binds the sockets and serves all 10,000 sites, even though no certificates are yet available. It performs the ACME operations in the background as quickly as possible with throttles and rate limits. As certificates are obtained, they are served, and errors are logged. Because this is an automated deployment, logs are monitored and errors cause alerts which the SRE team can fix.

In that last example, if the server were programmed with interactive behavior, an error with certificate #12 would probably return an error and stop running, causing all sites to stay down, even if all the other certificates would not have any errors. And even if there were no other errors, it would take the server hours to start up and serve any sites if it blocked until all certificates were obtained. On error, the process supervisor would then restart the process and, if misconfigured (as is common), would hammer the CA in a tight restart loop. This should be avoided, so a clear distinction between interactive and non-interactive modes is strongly recommended.


#### It should always be safe to reboot the machine or restart the process.

System reboots are often necessary for security updates. ACME software should try to minimize the risk that a happily-running application fails or takes a long time to come back up after a reboot. If rebooting is risky, that incentivizes people to delay security updates, which harms security.

So any time an ACME-capable web server is trying to come back up, if it already has certificates it should start up right away, even if those certificates are expired or close to expiring. That ensures that the server comes back as quickly as possible, making reboots cheap and low risk. It also minimizes the chance that one broken domain on a server with hundreds of domains can prevent the whole thing from coming back up after a reboot.


#### The most common reasons for failed ACME challenges are DNS, firewalls, and port contention.

When ACME challenges fail, it is a good idea to immediately question the configuration of relevant DNS and any applicable firewalls.

DNS can be tricky because an authoritative response depends on client perspective or the nuances of a specific DNS provider. DNS affects all challenge types, not just the DNS challenge. For the HTTP and TLS-ALPN challenges, A/AAAA records must be set to the public IP address of the machine that is solving the challenge. For the DNS challenge, a TXT record must be created in the domain's zone with a special token value for the duration of the challenge. The setting (and clearing) of this record varies from provider to provider and their user interfaces or APIs.

For the HTTP and TLS-ALPN challenges to succeed, ports 80 and 443 (respectively) on the public network interface must be open and either bound or forwarded to the client software that is solving the challenge. There is no way around this for these two challenges. If that is impossible, the DNS challenge can be used instead.

Even when ports 80/443 are configured properly, it is not uncommon for other software that is not ACME-enabled to be using ports 80 or 443 instead of the ACME client. Check to make sure that the correct ACME client is listening on those ports.


#### Write thorough and accurate documentation, and provide training as necessary.

Because certificates are security assets, users should understand how the certificate management software works in order to maintain security and privacy. Proper documentation goes a long way, and in some situations such as corporate/organizational settings, training is essential.

#### If an active certificate cannot be renewed in time, act according to your site's threat model.

If renewal errors persist to the end of the certificate's lifetime, there are several options:

- A) Use the expired certificate
- B) Use a self-signed certificate
- C) Serve connections without TLS
- D) Go offline

All these choices will result in error messages with respectable clients. So, your threat model should dictate what the best thing to do is. Assess what is causing renewal to fail for continuous days or weeks. It is important in all cases to send the right message to users.

If the reason for the failures is simply that the administrator changed their DNS provider account credentials but forgot to set the new credentials with their ACME client, then there is virtually no risk to users. This is truly a benign mistake, so choice A is probably the least disruptive and sends the lowest-risk signal.

If, however, DNS lookups are being hijacked in an ongoing attack, then the situation is more dangerous and choices B, C, or D should probably be used instead, depending on many factors including the nature of the site's content, political concerns, physical safety, user privacy, and much more.

These are not always easy decisions to make, but the vast majority of sites should probably just serve the expired certificate (option A). It is generally viewed as a low-risk error. Users with certain technical understanding will recognize this as a likely benign error and act accordingly.

An expired certificate is a direct consequence of the certificate not being renewed, so serving that expired certificate quickly exposes the problem rather than hiding the symptoms behind a closed socket or self-signed certificate. In addition, User-Agents can display a meaningful error message about the certificate, which allows the user to choose how to handle the situation. Sites that are urgently needed can still be accessed, and users who are not willing to take a slightly higher risk do not need to.

It is crucial that Web client software produce good and helpful error messages. For more discussion on instructive HTTPS errors, see [_After HTTPS: Indicating Risk Instead of Security_](https://scholarsarchive.byu.edu/etd/7403/).

For sites that have HSTS enabled, option C is not possible except on the first connection, so B or D may need to be used if option A is not desired. However, if enabled only through HTTP headers, HSTS can be bypassed on first access. Keep in mind that serving the site over HTTP (without TLS) puts both the user's privacy and the integrity of your site's content at risk.

Closing the socket entirely (option D) is only recommended if certificate errors are unacceptable and if option C puts the user or your site's integrity at too much risk. For example, sites that provide life-saving information during disasters or emergencies should probably stay online even if unencrypted.

Remember, these are extenuating circumstances. When making these decisions, always prioritize people's health and safety over pleasant user experiences.


#### Don’t make assumptions about URL structure.

Clients shouldn’t make any assumptions about the structure of account, order, authorization, challenge, or certificate resource URLs. RFC 8555 allows the client to discover the URLs of resources chosen by the server. Making assumptions about numeric components of URLs or their structure will result in the ACME client being incompatible with other RFC 8555 server implementations.


#### Don’t make assumptions about required authorizations.

The ACME server dictates what authorizations are required for a given order. The client should not make any assumptions about there being one authorization for each identifier, the order of authorization URLs within a created order resource, or other aspects of the authorization process. Instead, the client should create orders and respond dynamically to the authorizations returned in the order and their respective state.


### Open Questions

The following topics need more discussion and field testing as they do not yet have satisfying answers.

#### Can ACME account provisioning and legal agreements be proxied?

Often, a "subscriber" is managing certificates on behalf of a client. The client is often not present during the certificate obtain process. Can/should the vendor create the ACME account on their behalf and agree to the CA's legal terms on their behalf? Or proxy that agreement either implicitly or explicitly?

Additionally, if the legal terms change between renewals, can the software agree to the updated terms on behalf of the client without explicit approval? Getting explicit approval could be difficult/impossible and lead to downtime, not to mention high complexity to implement.


#### What are best practices for ACME account key rollover?

TODO.


#### How should CA-specific divergences from RFC 8555 be handled?

TODO.


#### What are some ways to avoid overfitting to Let's Encrypt so as to maximize CA interoperability?

TODO.

