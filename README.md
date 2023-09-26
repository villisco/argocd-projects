# argocd-projects

This is an EXAMPLE(!) ArgoCD __Projects__ (kind: AppProject) repository.\
The ArgoCD application named __"projects"__ (kind: Application) AUTO sync's these files!

`Kustomize` is used to patch the base files over with environment specific configs.\
For local testing get the `kustomize` tool from https://kustomize.io

> NB! __Only ArgoCD admins should be allowed to manage this repository!__

## Linked repositories

Splitting up the repositories provides option to manage permissions separately.

- https://github.com/villisco/argocd-setup - creates the __projects__ application that auto syncs this repository.
- https://github.com/villisco/argocd-apps - sync source for apps (kind: Application)


## Repository structure

```
argocd-projects
├── README.md
├── base
│   ├── kustomization.yaml
│   └── projects/
│       └── project1.yaml            <!--- shared part of the project manifest for all envs
└── overlays
    ├── dev                          <!--- kubernetes cluster
    │   ├── kustomization.yaml
    │   └── projects
    │       └── project1.yaml        <!--- patch over base project with env sepcific conf
    ├── live
    │   └── ...
    └── test
        └── ...
```

## Projects

Project is an logical way of grouping Applications together in ArgoCD.\
All Applications (__kind: Application__) must belong to an Project (__kind: AppProject__)!

__Projects__ provide following features:
- restrict what may be deployed (__trusted repositories__)
- restrict where apps may be deployed to (__destination clusters and namespaces__)
- __restrict__ what __kinds__ of __resources__ may or may not be deployed (e.g., RBAC, CRDs, DaemonSets, NetworkPolicy etc…)
- defining __project roles__ to provide application __RBAC__ (bound to OIDC groups and/or JWT tokens)
- define when application(s) are allowed to be synced with "__Sync Windows__"

Please define all new user __Applications__ in __argocd-apps__ repository!

## RBAC permissions

RBAC permission structure:
```
p, <role/user/group>, <resource>, <action>, <appproject>/<object>, allow|deny
```

> PS. Under project: `p, proj:<project-name>/<role-name>`

Possible __resources__:
```
clusters, projects, applications, applicationsets, repositories, certificates, accounts, gpgkeys, logs, exec, extensions
```

> NB! Roles under project inherit the restrictions configured to Project - you can not give permissions outside Projects allowed scope!

Possible __actions__:
```
get, create, update, delete, sync, override, action/<group/kind/action-name>
```

### Defining roles in Project

Define permissions under __role__ and __map role__ to an __group__.

Example:
```
apiVersion: argoproj.io/v1alpha1
kind: AppProject
  name: my-project
spec:
  roles:
    - name: read-only
      description: Read-only privileges to all apps in project
      policies:
        - p, proj:my-project:read-only, applications, get, my-project/*, allow
      groups:
        # add this group to users in keycloak
        - my-project_read-only
    - name: developer
      description: Developer privileges to all apps in project
      policies:
        # allow all app actions except delete & create
        - p, proj:my-project:developer, applications, *, my-project/*, allow
        - p, proj:my-project:developer, applications, delete, my-project/*, deny
        - p, proj:my-project:developer, applications, create, my-project/*, deny
        # allow viewing project
        - p, proj:my-project:developer, projects, get, my-project/*, allow
        # allow viewing projects repositories
        - p, proj:my-project:developer, repositories, get, my-project/*, allow
        # allow viewing pod logs
        - p, proj:my-project:developer, logs, get, my-project/*, allow
        # do not allow exec into pods
        - p, proj:my-project:developer, exec, create, my-project/*, deny
      groups:
        # add this group to users in keycloak
        - my-project_developer
```

> NB! After defining you can add groups (__my-project_read-only__, __my-project_developer__) to selected users in Keycloak "argocd" realm!

## Keycloak realm

Create same named ArgoCD groups there (see permissions example).\
Add users to groups to give them permissions in ArgoCD.

## ArgoCD docs

- https://argo-cd.readthedocs.io/en/stable/user-guide/projects/
- https://argo-cd.readthedocs.io/en/stable/user-guide/sync_windows/
- https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/
- https://argo-cd.readthedocs.io/en/latest/user-guide/commands/argocd_proj_role/
- https://github.com/argoproj/argo-cd/blob/master/assets/builtin-policy.csv (example permissions)