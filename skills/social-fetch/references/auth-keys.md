# Credentials and paid social sources

The Hosted Agent never accepts, reads, or configures social-provider API keys.
Company-paid services such as ScrapeCreators and Apify are available only
through their reviewed typed Magister actions; the Gateway owns the credential,
project binding, rate policy, billing, and health reporting.

If a typed service is unavailable, return `execution_unavailable`. Do not ask
the user to place a key in an environment variable, skill file, workspace file,
or chat message. A new provider requires a reviewed Gateway connection and
execution-contract update before this skill may use it.
