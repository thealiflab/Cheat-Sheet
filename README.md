# 📚 Cheat-Sheet

A growing collection of personal study and reference cheat sheets, written in Markdown. Each topic lives in its own folder so the sheets stay easy to browse and extend.

## Contents

| Topic | Cheat sheet | Description |
|-------|-------------|-------------|
| AWS | [AWS Certified Cloud Practitioner (CLF-C02)](AWS/aws-cloud-practitioner.md) | Complete exam study guide — exam overview, all four domains, critical service comparison tables, billing/pricing breakdown, initialisms, and a glossary, illustrated with official AWS service icons. |
| AWS | [Lightsail WordPress + SSL setup](AWS/lightsail-wp-instance-setup.md) | Step-by-step guide for provisioning a Lightsail WordPress instance and issuing a free Let's Encrypt certificate with Certbot, including Route 53 DNS, the www → apex redirect, `wp-config.php` fixes, and auto-renewal. |
| 3CX | [3CX Architecture Cheatsheet](3CX/3CX.md) | How a 3CX V20 phone system actually fits together — deployment models, core services, SIP vs. RTP, NAT traversal, and troubleshooting — aimed at whoever has to administer and troubleshoot it. |

## Repository structure

```
Cheat-Sheet/
├── 3CX/
│   └── 3CX.md
├── AWS/
│   ├── aws-cloud-practitioner.md
│   ├── lightsail-wp-instance-setup.md
│   └── assets/
└── README.md
```

## Usage

Open any `.md` file to read it directly on GitHub, or in a Markdown previewer (e.g. VS Code: `Ctrl+Shift+V`). The sheets are self-contained — no build step or dependencies.

## Contributing

To add a new cheat sheet, create a top-level `Topic-Area/` folder, add your Markdown file, and link it from the **Contents** table above.

## License & attribution

The AWS icons under `AWS/assets/` are provided by Amazon Web Services under the [AWS trademark guidelines](https://aws.amazon.com/trademark-guidelines/); see `AWS/assets/SOURCE.md` for details. All other content is for personal study use.
