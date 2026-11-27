# Help wanted

Lifeform is open to contributions. Below is the hardening work that would make it stronger, roughly in order of impact. Pick anything, comment on (or open) an issue so others know you are on it, and see [CONTRIBUTING.md](CONTRIBUTING.md) to get started.

Found a vulnerability instead? Please report it privately, see [SECURITY.md](SECURITY.md).


### Content Security Policy

**Good first issue.** The page should make no network requests. A strict CSP meta tag would guarantee it.

### Headless browser test

**Medium.** A Playwright test that opens the file, saves the offspring and checks the next generation is correct.
