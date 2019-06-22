# Stargate

Stargate is a generic CI/CD control plane based on Tekton and OPA.

Rules are written in [Rego](https://www.openpolicyagent.org/docs/latest/how-do-i-write-policies/) and hosted in an [Open Policy Agent](https://www.openpolicyagent.org/) instance.

Side-effects are delegated to a [Kubernetes](https://kubernetes.io/) cluster with [Tekton](https://github.com/tektoncd/pipeline) deployed.

The main goal is to make every use-case needed implementable as configuration without modifying Stargate itself.

The kinds of artifact you will need to create are : Rego rules, Tekton manifests and OCI images, hopefully the community will share all of them for most commons needs.

## Webhooks mode

Inputs are standardized HTTP requests (webhooks) from differents services (like Github/Gitlab/Slack/Trello/etc...).

Outputs are Tekton tasks/pipelines.

Example:
```
webhooks[task] {
    input.path == "/github"
    input.headers["X-Github-Event"][_] == "issues"
    input.payload.action == "opened"
    task := {
        "action": "gh-add-labels",
        "params": {
            "labels": ["need-approval"],
            "issue_id": input.payload.issue.id
        }
    }

}
```

![](https://i.imgur.com/vTfzA19.png)

## Cron mode

Another use-case of Stargate is to describe desired state as Rego policies, import current state as data (with a sidecar or cron job) and ask OPA to emit all needed tasks to converge.

Example:
```
violations[task] {
    member == data.desired.github.org[o].members[_]
    not data.current.github.org[o].members[member]
    task := {
        "action": "gh-add-org-member",
        "params": {
            "org_id": data.current.github.org[o].id,
            "member": member
        }
    }
}
```

Data are optionnals, you can embed rules parameters in policies.`

Example:
```
violations[task] {
    data.current.github.org[o].repo[r].protected_branchs[_] != "master"
    task := {
        "action": "gh-protect-branch",
        "params": {
            "org_id": data.current.github.org[o].id,
            "repo_id": data.current.github.org[o].repo[r].id,
            "branch": "master"
        }
    }
}
```
