# Start

docker compose --env-file $LOCALNET_DIR/compose.env \
               --env-file $LOCALNET_DIR/env/common.env \
               -f $LOCALNET_DIR/compose.yaml \
               -f $LOCALNET_DIR/resource-constraints.yaml \
               --profile sv \
               --profile app-provider \
               --profile app-user up -d

# Stop

docker compose --env-file $LOCALNET_DIR/compose.env \
               --env-file $LOCALNET_DIR/env/common.env \
               -f $LOCALNET_DIR/compose.yaml \
               -f $LOCALNET_DIR/resource-constraints.yaml \
               --profile sv \
               --profile app-provider \
               --profile app-user down -v

# Console

docker compose --env-file $LOCALNET_DIR/compose.env \
               --env-file $LOCALNET_DIR/env/common.env \
               -f $LOCALNET_DIR/compose.yaml \
               -f $LOCALNET_DIR/resource-constraints.yaml \
               run --rm console

# Bootstrap

bootstrap.synchronizer(
      synchronizerName = "appSynchronizer",
      sequencers = Seq(sequencer_app),
      mediators = Seq(mediator_app),
      synchronizerOwners = Seq(sequencer_app),
      synchronizerThreshold = 1,
      staticSynchronizerParameters = StaticSynchronizerParameters.defaultsWithoutKMS(ProtocolVersion.latest),
    )

participant_app_provider.synchronizers.connect_local(sequencer_app, "appSynchronizer")
participant_app_user.synchronizers.connect_local(sequencer_app, "appSynchronizer")

var appSynchronizerId = participant_app_provider.synchronizers.id_of("appSynchronizer")
var globalSynchronizerId = participant_app_provider.synchronizers.id_of("global")
participant_app_provider.dars.upload("dars/splitwell-0.1.16.dar", synchronizerId=appSynchronizerId)
participant_app_provider.dars.upload("dars/splitwell-0.1.16.dar", synchronizerId=globalSynchronizerId)
participant_app_user.dars.upload("dars/splitwell-0.1.16.dar", synchronizerId=appSynchronizerId)
participant_app_user.dars.upload("dars/splitwell-0.1.16.dar", synchronizerId=globalSynchronizerId)

# Configure Splitwell UIs

## Provider

docker exec -it splitwell-web-ui-app-provider /bin/bash

cd usr/share/nginx/html

cat <<'EOF' >config.js
const host = window.location.hostname;
window.splice_config = {
  auth: {
    algorithm: "hs-256-unsafe",
    secret: "unsafe",
    token_audience: "https://canton.network.global",
  },
  services: {
    wallet: {
      // URL of the web-ui, used to forward payment workflows to wallet
      uiUrl: window.location.origin.replace("splitwell", "wallet"),
    },
    splitwell: {
      // URL of the splitwell backend
      url: `${window.location.origin}/api/splitwell`,
    },
    jsonApi: {
      // URL of the JSON API for the participant
      url: window.location.origin.replace("splitwell", "json-ledger-api")+"/",
    },
    scan: {
      // URL of the scan app's HTTP API
      url: `http://scan.localhost:4000/api/scan`,
    },
  },
  spliceInstanceNames: {
    networkName: "Splice",
    networkFaviconUrl: "https://www.hyperledger.org/hubfs/hyperledgerfavicon.png",
    amuletName: "Amulet",
    amuletNameAcronym: "AMT",
    nameServiceName: "Amulet Name Service",
    nameServiceNameAcronym: "ANS",
  },
};
EOF

## User

docker exec -it splitwell-web-ui-app-user /bin/bash

cd usr/share/nginx/html

cat <<'EOF' >config.js
const host = window.location.hostname;
window.splice_config = {
  auth: {
    algorithm: "hs-256-unsafe",
    secret: "unsafe",
    token_audience: "https://canton.network.global",
  },
  services: {
    wallet: {
      // URL of the web-ui, used to forward payment workflows to wallet
      uiUrl: window.location.origin.replace("splitwell", "wallet"),
    },
    splitwell: {
      // URL of the splitwell backend
      url: "http://splitwell.localhost:3000/api/splitwell",
    },
    jsonApi: {
      // URL of the JSON API for the participant
      url: window.location.origin.replace("splitwell", "json-ledger-api")+"/",
    },
    scan: {
      // URL of the scan app's HTTP API
      url: `http://scan.localhost:4000/api/scan`,
    },
  },
  spliceInstanceNames: {
    networkName: "Splice",
    networkFaviconUrl: "https://www.hyperledger.org/hubfs/hyperledgerfavicon.png",
    amuletName: "Amulet",
    amuletNameAcronym: "AMT",
    nameServiceName: "Amulet Name Service",
    nameServiceNameAcronym: "ANS",
  },
};
EOF
