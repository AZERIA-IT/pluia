# Claude: custom connector (OAuth, no API key)

Steps from the Claude help center (individual Pro and Max plans; team plans have an
owner step first, see the link below):

1. Open **Customize > Connectors**.
2. Click **+ Add**, then **Add custom connector**.
3. Enter a name (for example `Pluia`) and the server URL: `https://pluia.fr/mcp`. Click **Continue**.
4. Review the authentication settings Claude detected, then click **Continue**.
5. Choose how to connect (for example **Sign in now**), sign in to your Pluia account on pluia.fr,
   review the requested access, and click **Authorize**.

You can remove the access at any time in your Pluia account (Applications connectées).

Source: https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp

Claude Code (terminal) can also use OAuth instead of an API key:

```bash
claude mcp add --transport http pluia https://pluia.fr/mcp
```

Then run `/mcp` inside Claude Code and authenticate the `pluia` server
(https://code.claude.com/docs/en/mcp).
