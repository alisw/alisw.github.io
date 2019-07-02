This is to document the release process of AliRoot / AliPhysics.

# Patching old ( <= AliRoot-v5-08) releases

Around `AliRoot-v5-08` / `AliRoot-v5-09` transition there
was a cleanup of large files, which were removed from the history of the official Github repository and left in a CERN hosted gitlab repository:

https://gitlab.cern.ch/alisw/AliRoot-legacy

For this reason any patch release on top of an old release should be tagged from that repository.
