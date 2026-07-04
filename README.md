# Nexus Repository Manager on DigitalOcean

Self-hosted an artifact repository from scratch: provisioned the server, secured it, configured access control, and published Java build artifacts to it from two different build tools, then verified everything through the REST API instead of just eyeballing the UI.

**Note on context:** This is hands-on lab work from the TechWorld with Nana DevOps bootcamp (Module 6). The two demo Java apps in this repo (`java-app`, `java-maven-app`) are course-provided scaffolding I used to exercise the publishing workflow, not original applications. I'd rather be upfront about that than have it look like more than it is. Everything else here, the server setup, the security configuration, the artifact management, I did myself and can walk through in an interview.

## Why This Exists

Every team that builds software needs somewhere to store the things it builds, compiled `.jar` files, Docker images, npm packages, with version history, access control, and cleanup policies so it doesn't grow forever. Nexus is one of the standard tools for that. This project was my way of understanding artifact management from the infrastructure side, not just consuming someone else's Artifactory instance.
