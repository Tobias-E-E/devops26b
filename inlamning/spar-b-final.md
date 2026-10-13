# Spår B – GitOps & Kubernetes: Flux mot Rahti (final)

- **Manifest-repo:** https://github.com/Tobias-E-E/devops26b-gitops
- **Dev-Route:** https://frontend-tobias-e-e-devops-dev.2.rahtiapp.fi/
- **Prod-Route:** https://frontend-tobias-e-e-devops-prod.2.rahtiapp.fi/

Solo-körning (testkörning av labben 2026-10-13).

## Finalen (steg 7)

- Prod-PR som pinnade sha:n: https://github.com/Tobias-E-E/devops26b-gitops/pull/3
  (`4616f40db8dd62b8b8cc4735427e5e424e387645` = merge-committen för
  https://github.com/Tobias-E-E/devops26b/pull/1, "Ny rubrik i frontend").
- Kodändringen live på prod-Routen: ![prod-Routen med ny rubrik](spar-b-07-prod-route.png) *(platshållare)*
- Dev behövde `oc rollout restart` för att `:latest` bara slås upp när en podd skapas, medan
  prod fick en ny tagg i podd-mallen via PR:en – en ändring Flux själv applicerar.

## Generalrepetitionen (steg 6)

- PR: https://github.com/Tobias-E-E/devops26b-gitops/pull/1 – `count` för backend 1 → 2 i `overlays/dev`.
  Revert: https://github.com/Tobias-E-E/devops26b-gitops/pull/2.
- Före/efter (terminalutdrag): se `spar-b-steg6-deployments.txt`.
- Tomma Actions-fliken i manifest-repot: ![tomma Actions](spar-b-06-actions-tom.png) *(platshållare)*

## Self-healing (steg 6.5)

Utskrift från `spar-b-steg6-self-healing.txt`:

```
$ oc scale deployment backend --replicas=4
deployment.apps/backend scaled
$ oc get deployments -w
12:09:21  backend    2/4     4            2           5m25s
12:09:36  backend    3/4     4            3           5m40s
12:09:41  backend    4/4     4            4           5m45s
12:09:47  backend    2/2     2            2           5m51s
```

Flux (kustomize-controller, som `flux-applier`) ändrade tillbaka den till det som står i git.

## Forbidden-meddelandet (steg 3)

Se `spar-b-steg3-forbidden.txt`: `User "system:serviceaccount:tobias-e-e-devops-dev:default" cannot patch resource "services"`.
Det var namespacets `default`-ServiceAccount som försökte skriva, inte jag.
![Forbidden](spar-b-03-forbidden.png) *(platshållare – textutdraget ovan är beviset)*

## Egen Role (steg 4)

Rollen `flux-applier` tillåter get/list/watch/create/update/patch/delete på Deployments, Services och
Routes – inte Secrets, ConfigMaps eller RoleBindings som `admin` ger. Se `spar-b-steg4-rbac.txt`.
![describe rolebinding + READY True](spar-b-04-rbac.png) *(platshållare)*

## Svar på frågorna

1. Flux kustomize-controller, som ServiceAccounten `flux-applier` i prod-namespacet (via `admin`-bindningen).
2. Flux kör inte som mig – den applicerar som den ServiceAccount Kustomization-objektet anger.
3. `default` används av alla poddar utan egen SA; rättigheter där läcker till hela namespacet.
4. En RoleBinding begränsar listan till ett namespace; en ClusterRoleBinding hade gällt hela klustret.
5. Bara Deployments/Services/Routes; en ConfigMap i `base/` ger Forbidden tills rollen får `configmaps`. Prod kunde gott ha den smala rollen.
6. `overlays/dev/kustomization.yaml` är Kustomizes byggrecept; `flux/dev/kustomization.yaml` är Flux-objektet som applicerar det.
7. Flux återskapar den vid nästa reconcile – git säger att den ska finnas.
8. `:latest` slås upp bara när en podd skapas; manifest-repot ändrades inte.
9. Flux hade strukit restart-annotationen och gjort en andra rollout; `replicas` står i git så `oc scale` ångras oavsett flagga.
10. `:latest` + restart är billigt men oprecist; sha-pin ger exakt svar på "vad kör prod" och rollback som en rad.
11. Risken att en nyckel in i produktionen läcker ur CI försvinner; i stället litar vi på git-historiken och RBAC i namespacet.
12. Fel i manifesten som renderar men är trasiga (t.ex. fel sha → ImagePullBackOff), och kvoten som tar slut mitt i en rollout.
