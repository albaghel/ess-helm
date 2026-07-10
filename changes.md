# changes

stuff changed in charts/matrix-stack/user_values/local.yaml

1. user directory search now returns everyone on the server, not just people you share a room with. added user_directory.enabled + search_all_users under synapse.additional.user-directory. before this, searching for someone you'd never been in a room with just gave nothing.

2. turned off e2ee by default for clients. added io.element.e2ee.force_disable: true to the wellKnownDelegation client config. element picks this up on startup and rooms come up unencrypted, no per-room toggle for users anymore.
