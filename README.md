# oci-arm-hunter

A scheduled GitHub Action that grabs an Oracle Cloud **Always Free ARM**
instance (`VM.Standard.A1.Flex`, 2 OCPU / 12 GB) in `ap-singapore-1` the moment
capacity frees up — running 24/7 on GitHub's servers so it works even when my
computer is off.

It polls every ~10 minutes, launches the instance when Singapore has room, then
stops (a guard skips launching once a `storetrack` instance exists) and opens an
issue to notify me.

Credentials (OCI API key + IDs) are stored as encrypted repository **secrets** —
nothing sensitive is in this code. This automates **my own** cloud account.

See the workflow in [.github/workflows/hunt.yml](.github/workflows/hunt.yml).
