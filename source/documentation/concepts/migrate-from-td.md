## Migrating to the Cloud Platform from Template Deploy

These are some of the things to think about when migrating your application from [Template Deploy][template-deploy] (TD) to the [Cloud Platform][cloud-platform] (CP).

### Production infrastructure

Any infrastructure resources, such as databases or cache servers, should be of the same size, and running the same software (e.g. postgres) version, for the CP version of your app. as for the TD version.

### Network traffic whitelisting

The IP numbers from which your app's traffic originates will change. If your app. talks to any external services which care about its source IP addresses, you may need to arrange for those services to whitelist the cluster IPs.

If you have whitelisted any IPs for inbound traffic, you will need to add the corresponding config to your kubernetes deploymnent files.

This section of the user guide describes both of these procedures: [IP whitelisting][ip-whitelist].

### DNS changes

When you are ready to switch from the TD version of your app. to the production version deployed in CP, the last step will be to change the [DNS] entries such that visitors to your app's domain are sent to the new CP version.

It's important to plan this step carefully. Some things to think about include:

* Who controls DNS for your app's domain (MoJ? GDS?). For `*.service.gov.uk` domains, changing the DNS requires action from GDS, and you will need to plan for this.
* Do you have users on private networks (e.g. DOM1, Quantum)? How confident are you that they will get the correct address after the DNS change? Some networks (e.g. Quantum), are known to have problems with DNS changes, so you may need to engage with the network operator (e.g. Atos, Vodafone) ahead of time.

### AWS access

All AWS resources in the Cloud Platform are in a separate, `moj-cp` AWS account. Access to this account is tightly restricted, so if you are accustomed to e.g. using the AWS web console to help operate your service, you will need to do things differently in the Cloud Platform.

### AWS region

Everything for the Cloud Platform is in the `eu-west-2` (London) region.  TD generally uses `eu-west-1` (Ireland). For features not available in the London region (eg SES, at time of writing), cross-region links can be created and will be evaluated on request.

### SSL

DNS and SSL are managed by cluster-shared services (cert-manager and external-dns)

 * Validation and deployment are managed by a single pipeline.
 * Renewals are automatic.

Does your service have users who are stuck with very old browsers? If so, consider the following:

* CP SSL certificates are generated by [LetsEncrypt]. If the user's browser does not recognise LetsEncrypt as a valid certificate authority, they may get security warnings when accessing your service, or they may not be permitted to access it at all.
* TLS version: Kubernetes supports a minimum TLS version of 1.2 If you have users with browsers that do not support TLS 1.2, they will not be able to access your service.

### Connections to/from service IPs

Inbound connections to service IPs are not possible, on the Cloud Platform. Access to your service must go via https to a URL, not by inbound access to an IP number.

Connections **to** resources outside the cluster are allowed, and pods will be NATed to the IPs of the VPC gateways (the IP addresses are pinned int the #ask-cloud-platform channel).

NB: UDP is not available at all.

### Application Architecture

1. Applications must log their entire output to STDOUT/STDERR.
   * A cluster-shared service (fluentd) automatically collects all the outputs and sends them to [ElasticSearch][kibana] with no explicit configuration needed.
   * Services that capture the output to re-format or send to a different destination can be run as multi-container pods, but may be superfluous.

1. You don't need to run an `init` service (e.g. `runit`, or `systemd`) in your containers. Kubernetes will keep your services running, by itself.

1. You don't need to run `cron` in your containers. Use [kubernetes cron jobs].

1. Keep startup as quick as possible
  * Initialisation tasks such as database migrations or loading seed data should be configured as separate [kubernetes jobs], not run as part of the initialisation of your main application. This is because, if your container takes too long to enter `ready` state, kubernetes will assume there's a problem and kill it.

1. Don't use local disk storage. If you need to write to the filesystem, use a persistent volume or, better still, an S3 bucket.

1. Monitoring is included with the Cloud Platform
   * The recommended solution is the combination of [Prometheus/AlertManager/Grafana][monitoring]
   * You can still use [Sentry][sentry]

1. Your application will be moved around, by kubernetes, from one worker node to another. i.e. your application should tolerate individual pods being killed and replaced.

1. Always run at least 3 replicas of your core, user-facing services, to ensure users do not experience downtime when kubernetes moves your pods to another node

### Additional features

1. A shared, highly-available deployment of reverse-proxy instances based on AWS ELB and Nginx runs in front of all application instances and offers features like [WAF][waf], custom [error pages][error-pages] or [IP-based ACLs][ip-whitelist] user-configurable per application but independent from its running state.
   * [Basic auth] can be quickly turned on/off at ingress level, useful when deploying dev services.

1. Several AWS resource types are tested and documented as [Terraform modules][terraform-modules]; they simplify deployment and default to best-practices options.

### Obsoleted features

1. Salt and CloudFormation are no longer used to define a deployment, replaced by any of Helm or Kubectl (see several [reference app][examples] examples).

1. Jenkins is no longer used to trigger deployments, teams are encouraged to [configure CircleCI][circleci] directly from their repositories.

1. For application logs, the path `awslogs` + CloudWatch + Lambda is replaced by `fluentd`.

1. Gandi and AWS ACM SSL certs are replaced by [Let's Encrypt](https://letsencrypt.org)


[template-deploy]: https://dsdmoj.atlassian.net/wiki/spaces/PLAT/pages/86310992/Template+Deploy
[cloud-platform]: https://github.com/ministryofjustice/cloud-platform
[terraform-modules]: https://user-guide.cloud-platform.service.justice.gov.uk/tasks.html#available-modules
[circleci]: https://user-guide.cloud-platform.service.justice.gov.uk/tasks.html#deploying-with-helm-and-circleci
[examples]: https://user-guide.cloud-platform.service.justice.gov.uk/tasks.html#deploying-a-39-hello-world-39-application-to-the-cloud-platform
[error-pages]: https://github.com/ministryofjustice/cloud-platform-custom-error-pages
[waf]: https://user-guide.cloud-platform.service.justice.gov.uk/tasks.html#modsecurity-web-application-firewall
[ip-whitelist]: https://user-guide.cloud-platform.service.justice.gov.uk/tasks.html#ip-whitelisting
[monitoring]: https://user-guide.cloud-platform.service.justice.gov.uk/tasks.html#monitoring-applications
[sentry]: https://sentry.service.dsd.io/mojds
[kibana]: https://kibana.cloud-platform.service.justice.gov.uk/_plugin/kibana
[DNS]: https://en.wikipedia.org/wiki/Domain_Name_System
[LetsEncrypt]: https://letsencrypt.org/
[kubernetes jobs]: https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/
[kubernetes cron jobs]: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
[Basic auth]: http://localhost:4567/tasks.html#add-http-basic-authentication