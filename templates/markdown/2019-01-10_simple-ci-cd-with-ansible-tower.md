## Simple CI/CD pipelines with Ansible Tower and Git

Let's say we have access to 3 different network environments:

- dev
- qa
- prod

<img width="100%" src="images/1-server-envs.png"/>

Let's say we have a web application we're working on and we want to:

- Develop on the application in a local development environment.
- Make a commit to the `master` branch of the application repository and have the **dev** server automatically build, test, and deploy the code in the `master` branch.
- When ready, manually merge from `master` to `qa` and have the **qa** server automatically build, test, and deploy the code in the `qa` branch.
- When ready, manually merge from `qa` to `prod` and have the **prod** server automatically build, test, and deploy the code in the `prod` branch.

<img width="100%" src="images/2-workflow.png"/>

Doing this we would be fulfilling some key software engineering best practices:

- Infrastructure as code
- Continuous Integration
- Continuous Delivery

Finally, let's say we want to do this with only [Ansible Tower](https://docs.ansible.com/?extIdCarryOver=true&sc_cid=701f2000001D8G5AAK) and [Git](https://git-scm.com/), and require the solution to be scalable to many teams, applications, etc.

Easy! Sam Doran wrote a great guide *Git Workflow for Ansible Development* (attached below) that explains how to utilize and link together Ansible and Git to create scalable development workflows. The following draws inspiration from the guide while putting it into practice using Ansible Tower and GitLab.

<img width="25%" src="images/3-tools.png"/>

### Git Repositories

Let's cover how we'll use Git and define how the Git repositories should be structured...

Let's define 3 different types of [repositories](https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository) and their [branches](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell):

- **ansible-inventory.git**: holds the `host-*` files and [host variables](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#host-variables) used by Ansible. There should be 1 host file per environment. Uses a single `master` branch. e.g.

   Branches:

         master

   Structure:

         $ tree ansible-inventory
         ├── group_vars/
         │   └── all.yml
         ├── hosts-dev
         ├── hosts-qa
         └── hosts-prod

- **ansible-playbooks.git**: holds the [playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html) that will run against the servers. Uses 1 branch per environment. These will be single playbooks that will need to run against multiple environments, it's important to parameterize the playbooks/[roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html) accordingly! **Note:** Roles should be their own repositories and only be included by these playbooks using [requirements.yml](https://docs.ansible.com/ansible/latest/reference_appendices/galaxy.html#installing-multiple-roles-from-a-file)

   Branches:

         master (dev)
         qa
         prod

   Structure:

         $ tree ansible-playbooks
         ├── README.md
         ├── ansible.cfg
         ├── pb-ping.yml
         ├── pb-reload-app.yml
         ├── pb-setup-app.yml
         ├── pb-setup-system.yml
         ├── pb-test-app.yml
         └── templates
             └── docker-compose.yml.j2

- **application-repo.git**: holds the application code that will get deployed. Uses 1 branch per environment. This will be one code base that will need to run in multiple environments, it's important to parameterize the application code accordingly!

   Branches:

         master (dev)
         qa
         prod

   Structure:


         $ tree -d -L 1 application-repo
         ├── app
         ├── bin
         ├── config
         ├── discordbot
         ├── docs
         ├── lib
         ├── log
         ├── public
         ├── test
         ├── tmp
         └── vendor

```sh
# Git Repositories
ansible-inventory.git
ansible-playbooks.git
application-repo.git
```

This could scale out to multiple teams and applications easily. Each team would simply own their own versions of these repos and could have one or many of each.

<img width="100%" src="images/4-repos.png"/>

#### Branching

Here's how to visualize the branching structure for `ansible-playbooks.git` and `application-repo.git`. The image visualizes manually merging code into `master` from a `feature-branch`, in this guide though, we're allowing commits directly to `master`. Either approach is completely valid.

<img width="50%" src="images/git-branching.png"/>

### Tower Resources

#### Projects

- 1 [project](https://docs.ansible.com/ansible-tower/latest/html/userguide/projects.html) for each inventory repo. e.g. `inventory` project -> `ansible-inventory.git` repo.
- 1 project for each **branch** of the playbook repo. e.g. `playbooks-dev`, `playbooks-qa`, `playbooks-prod` projects -> `ansible-playbooks.git` repo.

         # Tower Projects
         ansible-inventory # points to ansible-inventory.git:master
         playbooks-dev # points to ansible-playbooks.git:master
         playbooks-qa # points to ansible-playbooks.git:qa
         playbooks-prod # points to ansible-playbooks.git:prod

#### Inventories

- 1 [inventory](https://docs.ansible.com/ansible-tower/latest/html/userguide/inventories.html) per environment, with each having a project source the points to the `inventory` project and the correct `hosts-*` file.

         # Tower Inventories
         hosts-dev # inventory project source of hosts-dev in ansible-inventory.git repo
         hosts-qa # inventory project source of hosts-qa in ansible-inventory.git repo
         hosts-prod # inventory project source of hosts-prod in ansible-inventory.git repo

#### Job Templates

- [Job Templates](https://docs.ansible.com/ansible-tower/latest/html/userguide/job_templates.html) in Ansible Tower essentially are pre-configured playbook runs pointing to hosts files and having environment specific [variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html) passed in. Job templates will be created with the intention of running against a specific environment. So it is needed to have 1 job template per playbook, per environment. e.g. (using the playbooks in the `ansible-playbooks.git` above)

         # Job Templates
         job-setup-app-dev # uses (playbooks-dev, hosts-dev)
         job-test-app-dev # uses (playbooks-dev, hosts-dev)
         job-reload-app-dev # uses (playbooks-dev, hosts-dev)

         job-setup-app-qa # uses (playbooks-qa, hosts-qa)
         job-test-app-qa # uses (playbooks-qa, hosts-qa)
         job-reload-app-qa # uses (playbooks-qa, hosts-qa)

         job-setup-app-prod # uses (playbooks-prod, hosts-prod)
         job-test-app-prod # uses (playbooks-prod, hosts-prod)
         job-reload-app-prod # uses (playbooks-prod, hosts-prod)

#### Workflows

- Workflows will act against specific environments. They should have the intended environment appended to the name, and will only use job templates from the same environment.

         # Tower Workflows
         workflow-deploy-app-dev # job-setup-app-dev -> job-test-app-dev -> job-reload-app-dev

         workflow-deploy-app-qa # job-setup-app-qa -> job-test-app-qa -> job-reload-app-qa

         workflow-deploy-app-prod # job-setup-app-prod -> job-test-app-prod -> job-reload-app-prod

#### Scaling

Managing and creating all these resources is easily scalable using Ansible Tower.

As an admin, create a new organization and assign a user(s) as an organization admin. This user(s) will be able to provision their own Inventories, Projects, Job Templates, Workflows, etc.

Create a new organization per team and you'll achieve multi-tenancy, while still allowing organizations/teams to share resources as needed.

A lot of text? Here is a visual representation of what we're accomplishing:

<img width="100%" src="images/4-tower-resources.png"/>

### Integration between Git and Ansible Tower

Now that we have all the resources for Git and Ansible Tower mapped out, it's time to set up the integration between the two. For this specific implementation, we're going to use [GitLab](https://about.gitlab.com/) and take advantage of their [webhooks](https://docs.gitlab.com/ee/user/project/integrations/webhooks.html) feature, which allows us to send HTTP calls when something happens in a repository. All major Git SaaS solutions offer similar features ([GitHub](https://github.com/), [BitBucket](https://bitbucket.org/), GitLab, etc). Though you could also accomplish the same thing with just bare Git repositories using [hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks). GitLab is free and open source, making it easy to install and use, and all the following examples will be based off it.

#### Ansible Tower application and token configuration

Within Tower, we'll need to create an [Application](https://docs.ansible.com/ansible-tower/latest/html/userguide/applications_auth.html) that will have an `Authorization Grant Type: Resource owner password-based`. Once this is configured, we can use it to create an [application token](https://docs.ansible.com/ansible-tower/latest/html/administration/oauth2_token_auth.html).

From the UI:

`Administration -> Applications -> Create`

Next is to create a token with write access so that this token can be used to create/start Ansible Tower resources through the [API](https://docs.ansible.com/ansible-tower/latest/html/towerapi/index.html).

From the UI:

`Users -> <your_user> -> Tokens -> Create`

Once this token is created, Tower will give us a unique token ID e.g. `mwS4G54Xr2En87RDdHtp7UxLDexcnK`

We can then use this token when making API calls against Tower. The token will have write privileges to any resource that the user it was created under has.

You could create a special service-account user, give it access to only the workflows/playbooks needed, and create the token under that user to easily manage the token's access.

#### GitLab integrations/webhooks configuration

Again, here are the goals:

1. When code is committed to `master` branch of the `application-repo.git` then have a webhook call the Tower API to launch workflow: `workflow-deploy-app-dev`

2. When a manual merge from `master` to `qa` is performed then have a webhook call the Tower API to launch workflow: `workflow-deploy-app-qa`

3. When a manual merge from `qa` to `prod` is performed then have a webhook call the Tower API to launch workflow: `workflow-deploy-app-prod`

The common case would be only commits to `master` are allowed, and merge requests into `qa` and `prod` would be manually approved, or could happen on a certain cadence.

All workflows are configured with 2 main steps (yours will vary).

1. Pull the latest code from the relevant branch (`master`, `qa`, `prod`), and run application tests. **ONLY IF SUCCESSFULL** -> go to step 2.
2. Reload the running application with the latest code from the relevant branch (`master`, `qa`, `prod`)

Basically, we need to configure the GitLab webhooks to call the correct workflows.

From the UI:

`Repository -> Settings -> Integrations -> Create`

```yaml
url: https://tower.mydomain.com/api/v2/workflow_job_templates/<id>/launch/
secret token: <above token from Tower>
Push events: <branch>
```

In the application repository settings, create an integration per branch/environment.

**Note** GitLab sends the secret token in an HTTP header as: `X-Gitlab-Token: <token>` in `/opt/gitlab/embedded/service/gitlab-rails/app/services/web_hook_service.rb`. Tower expects the Token sent in an HTTP header as: `Authorization: Bearer <token>`. It was necessary to edit the GitLab source code to achieve this. The correct way would be to write a custom "Project Service" GitLab integratation with Tower, but the source code edit was quick and easy :). See: https://github.com/gitlabhq/gitlabhq/blob/64fabd5dc132b7988d83bddb6f17a16223e76508/app/services/web_hook_service.rb#L116

Once the GitLab webhooks are working, everything is in place.

<img width="100%" src="images/5-final-workflow.png"/>

#### Conclusion

Now we're able to have:

- One application code base
- A single repository for Ansible Playbooks to provision the servers and provision/test/redeploy our application.

Which allows us the ability to:

- Automated testing and deployment of code once committed.
- Commit application code once and move it into multiple environments using automation.
- Run separate branches of application AND playbook repos in each environment.

The above ensures:

- Infrastructure as code
- Continuous integration
- Continuous deployment

Using only

- Ansible Tower
- Git
