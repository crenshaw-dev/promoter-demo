# helm-guestbook example

The base files for this example are taken from here: https://github.com/argoproj/argocd-example-apps/tree/master/helm-guestbook

We've added some values files to provide environment-specific "overlays".

We've also added .argocd-source-<app name>.yaml files to provide app-specific overlays.

The point of the .argocd-source files is to ensure that _all_ information which can affect manifest hydration is
represented in the git repo.
