# Project Directory Structure

```text
PROJECT-PRISMA-PRESS/
├── .firebase/                  # (Hidden slightly at the bottom / standard config)
├── dist/                       # Compiled JavaScript output
├── generated/                  # Generated Prisma client engines
├── node_modules/               # Installed project dependencies
├── prisma/                     # Database schema configurations
└── src/                        # Main source code directory
    ├── config/                 # App configuration variables 
    ├── lib/                    # Third-party service initializations
    ├── middlewares/            # Custom express routing middlewares
    ├── modules/                # Feature-based module architecture
    │   ├── auth/               # Authentication handlers
    │   ├── comments/           # Comment handlers
    │   ├── post/               # Post handlers
    │   ├── premium/            # Premium feature handlers
    │   │   ├── premium.router.ts
    │   │   ├── premium.service.ts
    │   │   └── premium.controller.ts
    │   ├── subscription/       # Subscription handlers
    │   │   ├── subscription.controller.ts
    │   │   ├── subscription.route.ts
    │   │   ├── subscription.service.ts
    │   │   └── subscription.utils.ts
    │   └── users/              # User account profiles handling
    ├── utils/                  # Shared helper utility files
    ├── app.ts                  # App instantiation & middleware loader
    └── server.ts               # Core server entry point listener
├── .env                        # Local environment private variables (git-ignored)
├── .env.example                # Shared safe environment structure template
├── .gitignore                  # Instructs git on files to completely ignore
├── package-lock.json           # Secure version lock state tree
├── package.json                # Custom scripts and main project configurations
└── Prisma-Press-Backend-Requirement-Analysis.pdf
