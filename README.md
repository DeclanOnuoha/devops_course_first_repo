# Build-It-Yourself Tutorial: Professional Git Workflow Edition

Same framework as before, but built the way a real team does it: **CI is wired up on `main` before a single test exists**, and every piece of work after that goes: `branch → commit → push → PR → CI runs → merge → pull main → delete branch`. You never commit directly to `main`.

**Target app:** [saucedemo.com](https://www.saucedemo.com/)

---

## Prerequisites

- Git installed, GitHub account
- Python 3.10+, Chrome
- Optional but recommended: [GitHub CLI](https://cli.github.com/) (`gh`) — lets you open/merge PRs from the terminal instead of switching to the browser. Everything below shows both the `gh` command and the web-UI equivalent.

---

## Phase 0 — Repo + CI skeleton on `main` (before any tests exist)

This is the part people skip and shouldn't. Get the pipeline green on an empty project first, so every feature branch after this is validated by a pipeline that's already proven to work.

### 0.1 Create the GitHub repo

```bash
gh repo create robot-ecommerce-framework --public --clone
cd robot-ecommerce-framework
```
*(No `gh`? Create the repo on github.com manually, then:)*
```bash
git clone https://github.com/YOUR_USERNAME/robot-ecommerce-framework.git
cd robot-ecommerce-framework
```

### 0.2 Scaffold the project

```bash
mkdir -p tests resources/page_objects data .github/workflows results
python -m venv venv
source venv/bin/activate   # venv\Scripts\activate on Windows
pip install robotframework robotframework-seleniumlibrary selenium
pip freeze > requirements.txt
```

### 0.3 `.gitignore`

```
venv/
__pycache__/
*.pyc
results/
.DS_Store
```

### 0.4 A minimal smoke test — CI needs *something* to run

`tests/smoke_check.robot`:
```robotframework
*** Settings ***
Library    SeleniumLibrary

*** Variables ***
${URL}    https://www.saucedemo.com/

*** Test Cases ***
Site Is Reachable
    [Tags]    smoke
    Open Browser    ${URL}    chrome    options=add_argument("--headless=new")
    Title Should Be    Swag Labs
    [Teardown]    Close Browser
```

This single test is the seed that proves the whole pipeline — browser launch, headless mode, assertions — works end to end before you build anything real on top of it.

### 0.5 The CI workflow itself

`.github/workflows/ci.yml`:
```yaml
name: Robot Framework Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: robot --variable HEADLESS:True --outputdir results tests/
      - if: always()
        uses: actions/upload-artifact@v4
        with:
          name: robot-test-results
          path: results/
```

Note it triggers on **both** `push` to `main` and `pull_request` into `main` — that second trigger is what makes CI run on every PR before you're allowed to merge.

### 0.6 Commit the skeleton straight to `main`

This is the *one and only* time you commit directly to `main` — establishing the baseline before branch protection goes on.

```bash
git add .
git commit -m "chore: project skeleton with CI pipeline and smoke test"
git push -u origin main
```

Go to the **Actions** tab on GitHub. Watch it go green. If it fails, fix it now — don't build features on top of a broken pipeline.

### 0.7 Protect `main`

Now lock the branch so nothing merges without passing CI and going through a PR:

- GitHub web UI: **Settings → Branches → Add branch protection rule**
  - Branch name pattern: `main`
  - Check **"Require a pull request before merging"**
  - Check **"Require status checks to pass before merging"** → select the `test` job (it'll only appear in the list after CI has run at least once, which is why 0.6 comes first)
  - Save

```bash
gh api repos/YOUR_USERNAME/robot-ecommerce-framework/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["test"]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":0}' \
  --field restrictions=null
```

From this point on, `git push origin main` directly will be rejected. Everything goes through a branch.

---

## The workflow you'll repeat for every feature

```bash
git checkout main
git pull origin main
git checkout -b feature/<name>
# ... do work, commit ...
git push -u origin feature/<name>
gh pr create --fill          # or open the PR on github.com
# wait for the green check on the PR
gh pr merge --merge --delete-branch    # or click "Merge pull request" on github.com
git checkout main
git pull origin main
```

You'll run this exact loop 4 times below. Once it's muscle memory, that's the real skill this project teaches — not just the Robot Framework syntax.

---

## Phase 1 — feature/login-tests

```bash
git checkout -b feature/login-tests
```

Create `data/variables.py`:
```python
USERS = {
    "standard": {"username": "standard_user", "password": "secret_sauce"},
    "locked_out": {"username": "locked_out_user", "password": "secret_sauce"},
    "invalid_password": {"username": "standard_user", "password": "wrong_password"},
}

CHECKOUT_INFO = {
    "first_name": "Jane",
    "last_name": "Doe",
    "postal_code": "00100",
}
```

Create `resources/common.resource`:
```robotframework
*** Settings ***
Library      SeleniumLibrary
Variables     ${CURDIR}/../data/variables.py

*** Variables ***
${BROWSER}      chrome
${URL}          https://www.saucedemo.com/
${TIMEOUT}      10s
${HEADLESS}     False

*** Keywords ***
Open Application
    ${options}=    Get Chrome Options
    Open Browser    ${URL}    ${BROWSER}    options=${options}
    Maximize Browser Window
    Set Selenium Timeout    ${TIMEOUT}

Close Application
    Close All Browsers

Go To Login Page
    Delete All Cookies
    Go To    ${URL}

Get Chrome Options
    ${options}=    Evaluate    selenium.webdriver.ChromeOptions()    modules=selenium.webdriver
    Run Keyword If    '${HEADLESS}' == 'True'    Set Headless Chrome Options    ${options}
    RETURN    ${options}

Set Headless Chrome Options
    [Arguments]    ${options}
    Call Method    ${options}    add_argument    --headless=new
    Call Method    ${options}    add_argument    --window-size=1920,1080
```

Create `resources/page_objects/login_page.resource`:
```robotframework
*** Settings ***
Library    SeleniumLibrary

*** Variables ***
${USERNAME_FIELD}    id:user-name
${PASSWORD_FIELD}    id:password
${LOGIN_BUTTON}       id:login-button
${ERROR_MESSAGE}      css:h3[data-test="error"]

*** Keywords ***
Login As
    [Arguments]    ${username}    ${password}
    Input Text      ${USERNAME_FIELD}    ${username}
    Input Password    ${PASSWORD_FIELD}    ${password}
    Click Button    ${LOGIN_BUTTON}

Login Error Message Should Be
    [Arguments]    ${expected_message}
    Wait Until Element Is Visible    ${ERROR_MESSAGE}
    Element Text Should Be    ${ERROR_MESSAGE}    ${expected_message}
```

Create `resources/page_objects/inventory_page.resource`:
```robotframework
*** Settings ***
Library    SeleniumLibrary

*** Variables ***
${INVENTORY_CONTAINER}    id:inventory_container

*** Keywords ***
Inventory Page Should Be Open
    Wait Until Element Is Visible    ${INVENTORY_CONTAINER}
    Location Should Contain    inventory.html
```

Create `tests/login_tests.robot`:
```robotframework
*** Settings ***
Resource        ../resources/common.resource
Resource        ../resources/page_objects/login_page.resource
Resource        ../resources/page_objects/inventory_page.resource
Suite Setup      Open Application
Suite Teardown    Close Application
Test Teardown    Go To Login Page

*** Test Cases ***
Valid Login Should Land On Inventory Page
    [Tags]    login    smoke
    Login As    ${USERS}[standard][username]    ${USERS}[standard][password]
    Inventory Page Should Be Open

Invalid Password Shows Error Message
    [Tags]    login    negative
    Login As    ${USERS}[invalid_password][username]    ${USERS}[invalid_password][password]
    Login Error Message Should Be
    ...    Epic sadface: Username and password do not match any user in this service

Locked Out User Cannot Log In
    [Tags]    login    negative
    Login As    ${USERS}[locked_out][username]    ${USERS}[locked_out][password]
    Login Error Message Should Be    Epic sadface: Sorry, this user has been locked out.
```

**Run locally before pushing anything — never rely on CI to tell you your first draft is broken:**
```bash
robot --dryrun --outputdir results tests/
robot --outputdir results tests/login_tests.robot
```
4 tests passing (3 login + the smoke check).

Commit and open the PR:
```bash
git add .
git commit -m "feat: add login page object and login test suite"
git push -u origin feature/login-tests
gh pr create --title "Add login tests" --body "Adds login page object + 3 login scenarios (valid, invalid password, locked out)."
```

Go watch the check run on the PR page. Green check → merge:
```bash
gh pr merge --merge --delete-branch
git checkout main
git pull origin main
```

---

## Phase 2 — feature/inventory-tests

```bash
git checkout -b feature/inventory-tests
```

Expand `resources/page_objects/inventory_page.resource` to add:
```robotframework
*** Settings ***
Library    SeleniumLibrary
Library    String

*** Variables ***
${INVENTORY_CONTAINER}     id:inventory_container
${CART_ICON}                 css:.shopping_cart_link
${CART_BADGE}                css:.shopping_cart_badge
${SORT_DROPDOWN}            id:product_sort_container
${PRODUCT_PRICE_LIST}       css:.inventory_item_price
${ADD_TO_CART_BACKPACK}     id:add-to-cart-sauce-labs-backpack

*** Keywords ***
Inventory Page Should Be Open
    Wait Until Element Is Visible    ${INVENTORY_CONTAINER}
    Location Should Contain    inventory.html

Add Product To Cart By Locator
    [Arguments]    ${product_button_locator}
    Click Element    ${product_button_locator}

Cart Badge Count Should Be
    [Arguments]    ${expected_count}
    Wait Until Element Is Visible    ${CART_BADGE}
    Element Text Should Be    ${CART_BADGE}    ${expected_count}

Cart Badge Should Not Be Visible
    Element Should Not Be Visible    ${CART_BADGE}

Sort Products By
    [Arguments]    ${option_label}
    Select From List By Label    ${SORT_DROPDOWN}    ${option_label}

Get All Product Prices
    ${price_elements}=    Get WebElements    ${PRODUCT_PRICE_LIST}
    ${prices}=    Create List
    FOR    ${element}    IN    @{price_elements}
        ${text}=    Get Text    ${element}
        ${clean}=    Remove String    ${text}    $
        ${as_float}=    Convert To Number    ${clean}
        Append To List    ${prices}    ${as_float}
    END
    RETURN    ${prices}

Go To Cart
    Click Element    ${CART_ICON}
```

⚠️ Note `Library String` — needed for `Remove String`. Forgetting it is the single most common break in this build; `--dryrun` catches it instantly.

Create `tests/inventory_tests.robot`:
```robotframework
*** Settings ***
Resource        ../resources/common.resource
Resource        ../resources/page_objects/login_page.resource
Resource        ../resources/page_objects/inventory_page.resource
Suite Setup      Log In As Standard User
Suite Teardown    Close Application

*** Test Cases ***
Cart Badge Is Hidden When Cart Is Empty
    [Tags]    inventory    smoke
    Cart Badge Should Not Be Visible

Adding A Product Updates The Cart Badge
    [Tags]    inventory    smoke
    Add Product To Cart By Locator    ${ADD_TO_CART_BACKPACK}
    Cart Badge Count Should Be    1
    [Teardown]    Add Product To Cart By Locator    ${ADD_TO_CART_BACKPACK}

Products Can Be Sorted By Price Low To High
    [Tags]    inventory    regression
    Sort Products By    Price (low to high)
    ${prices}=    Get All Product Prices
    ${sorted_prices}=    Evaluate    sorted($prices)
    Lists Should Be Equal    ${prices}    ${sorted_prices}


*** Keywords ***
Log In As Standard User
    Open Application
    Login As    ${USERS}[standard][username]    ${USERS}[standard][password]
    Inventory Page Should Be Open
```

Validate, commit, PR, merge:
```bash
robot --dryrun --outputdir results tests/
robot --outputdir results tests/inventory_tests.robot

git add .
git commit -m "feat: add inventory page object and cart/sorting tests"
git push -u origin feature/inventory-tests
gh pr create --title "Add inventory tests" --body "Adds cart badge state tests and price-sort verification."
# wait for green check
gh pr merge --merge --delete-branch
git checkout main
git pull origin main
```

---

## Phase 3 — feature/checkout-tests

```bash
git checkout -b feature/checkout-tests
```

Create `resources/page_objects/cart_page.resource`:
```robotframework
*** Settings ***
Library    SeleniumLibrary

*** Variables ***
${CART_LIST}            id:cart_list
${CART_ITEM_NAME}       css:.cart_item .inventory_item_name
${CHECKOUT_BUTTON}      id:checkout

*** Keywords ***
Cart Page Should Be Open
    Wait Until Element Is Visible    ${CART_LIST}
    Location Should Contain    cart.html

Cart Should Contain Product
    [Arguments]    ${product_name}
    Wait Until Page Contains Element    ${CART_ITEM_NAME}
    Page Should Contain Element    xpath://div[@class="inventory_item_name" and text()="${product_name}"]

Proceed To Checkout
    Click Element    ${CHECKOUT_BUTTON}
```

Create `resources/page_objects/checkout_page.resource`:
```robotframework
*** Settings ***
Library    SeleniumLibrary

*** Variables ***
${FIRST_NAME_FIELD}      id:first-name
${LAST_NAME_FIELD}       id:last-name
${POSTAL_CODE_FIELD}     id:postal-code
${CONTINUE_BUTTON}       id:continue
${FINISH_BUTTON}         id:finish
${COMPLETE_HEADER}       css:.complete-header
${ERROR_MESSAGE}         css:h3[data-test="error"]

*** Keywords ***
Fill Checkout Information
    [Arguments]    ${first_name}    ${last_name}    ${postal_code}
    Input Text    ${FIRST_NAME_FIELD}    ${first_name}
    Input Text    ${LAST_NAME_FIELD}    ${last_name}
    Input Text    ${POSTAL_CODE_FIELD}    ${postal_code}
    Click Element    ${CONTINUE_BUTTON}

Checkout Error Should Be
    [Arguments]    ${expected_message}
    Wait Until Element Is Visible    ${ERROR_MESSAGE}
    Element Text Should Be    ${ERROR_MESSAGE}    ${expected_message}

Finish Checkout
    Click Element    ${FINISH_BUTTON}

Order Should Be Complete
    Wait Until Element Is Visible    ${COMPLETE_HEADER}
    Element Text Should Be    ${COMPLETE_HEADER}    Thank you for your order!
```

Create `tests/checkout_tests.robot`:
```robotframework
*** Settings ***
Resource        ../resources/common.resource
Resource        ../resources/page_objects/login_page.resource
Resource        ../resources/page_objects/inventory_page.resource
Resource        ../resources/page_objects/cart_page.resource
Resource        ../resources/page_objects/checkout_page.resource
Suite Setup      Open Application
Suite Teardown    Close Application
Test Setup       Log In And Add Backpack To Cart

*** Test Cases ***
Complete Checkout With Valid Information
    [Tags]    checkout    e2e    smoke
    Go To Cart
    Cart Page Should Be Open
    Cart Should Contain Product    Sauce Labs Backpack
    Proceed To Checkout
    Fill Checkout Information    ${CHECKOUT_INFO}[first_name]    ${CHECKOUT_INFO}[last_name]    ${CHECKOUT_INFO}[postal_code]
    Finish Checkout
    Order Should Be Complete

Checkout Requires First Name
    [Tags]    checkout    negative
    Go To Cart
    Proceed To Checkout
    Fill Checkout Information    ${EMPTY}    ${CHECKOUT_INFO}[last_name]    ${CHECKOUT_INFO}[postal_code]
    Checkout Error Should Be    Error: First Name is required


*** Keywords ***
Log In And Add Backpack To Cart
    Go To Login Page
    Login As    ${USERS}[standard][username]    ${USERS}[standard][password]
    Inventory Page Should Be Open
    Add Product To Cart By Locator    ${ADD_TO_CART_BACKPACK}
```

Validate, commit, PR, merge:
```bash
robot --dryrun --outputdir results tests/
robot --outputdir results tests/checkout_tests.robot

git add .
git commit -m "feat: add cart and checkout page objects with e2e purchase flow"
git push -u origin feature/checkout-tests
gh pr create --title "Add checkout tests" --body "Adds full purchase flow (login -> cart -> checkout -> confirmation) plus form validation."
gh pr merge --merge --delete-branch
git checkout main
git pull origin main
```

---

## Phase 4 — feature/cleanup

Retire the throwaway smoke check now that real tests exist and prove the same thing plus more:

```bash
git checkout -b feature/cleanup
git rm tests/smoke_check.robot
```

Add `robot.toml` so RobotCode and the CLI share config:
```toml
[tool.robotcode]
args = ["--outputdir", "results"]

[tool.robotcode.robot]
paths = ["tests"]
```

```bash
robot --dryrun --outputdir results tests/
robot --outputdir results tests/

git add .
git commit -m "chore: remove throwaway smoke test, add robotcode config"
git push -u origin feature/cleanup
gh pr create --title "Cleanup: remove placeholder smoke test" --body "Real suites now cover what the placeholder proved. Adds robot.toml for editor/CLI parity."
gh pr merge --merge --delete-branch
git checkout main
git pull origin main
```

---

## Where you land

`main` now has, entirely through reviewed, CI-gated PRs:
- 3 test suites, 8 real tests (login, inventory, checkout) — no throwaway code left behind
- Branch protection preventing any direct push or unreviewed merge
- A CI badge you can drop in a README:
  ```markdown
  [![Robot Framework Tests](https://github.com/YOUR_USERNAME/robot-ecommerce-framework/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR_USERNAME/robot-ecommerce-framework/actions)
  ```
- A commit history that reads like actual feature work (`feat:`, `chore:` prefixes), not one giant dump commit

This is the part of the portfolio that's easy to skip and hardest to fake convincingly in an interview — being able to say "every one of these merged through CI on a protected branch" is a stronger signal than the test code itself.

Want me to add a `feature/pr-template` phase that adds a `.github/pull_request_template.md` and a CODEOWNERS file, to round out the "this looked like a real team repo" picture?
