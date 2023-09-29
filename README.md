# Steps and lessons learned for the framework Reflex

## Useful Links

- [Reflex](https://reflex.dev)

  - [Docs](https://reflex.dev/docs/getting-started/introduction/)

  - [GitHub Profile](https://github.com/reflex-dev)

  - [Main GitHub Repo](https://github.com/reflex-dev/reflex)

  - [Helpful Examples GitHub Repo](https://github.com/reflex-dev/reflex-examples)

  - [Example Used](https://github.com/reflex-dev/reflex-examples/tree/main/crm)

### Important Info

- Python Framework

- Uses [Next.JS](https://nextjs.org)

- Uses [SQLModel](https://sqlmodel.tiangolo.com) as ORM

## Steps

### Chapter 1 - Create and init a new project

> Inside the terminal

1. Go to desktop

```bash
~ $ cd Desktop/
```

2. Create new folder

```bash
~ $ mkdir appname
```

3. Go into the new folder

```bash
~ $ cd appname/
```

4. Create new virtual environment

```bash
appname $ python3 -m venv venv
```

5. Activate virtual environment

```bash
source venv/bin/activate
```

### Chapter 2 - (venv) is activated install reflex

> Should see (venv) in the terminal

6. Install reflex

```bash
appname $ pip install reflex
```

7. Initialize reflex

```bash
appname $ reflex init
```

8. Check files (optional)

```bash
appname $ ls
```

> Confirm you have at least these 4 folders: assets, appname, rxconfig.py, venv

### Chapter 3 -  Inside the codebase - Milestone

> If everything has gone right you should see this

```bash
# Tree structure
> __pycahce__
> appname
> assets
rxconfig.py
> venv
```

### Chapter 4 - Run the app

9. Run the app

```bash
appname $ reflex run
# Should see "App running at: http://localhost:3000"
```

> Run this command to see if the app is working with no bugs or errors
>
> To stop the app press "Ctrl + C"

### Chapter 5 - State folder and files

> Important folders: state
>
> Important files: models.py, **init**.py, state.py

10. Create state folder with files:
[
  models.py,
  state.py,
  login.py
  **init**.py,
  ].
Matching below

```bash
> appname 
  > state
    - __init__.py
    - login.py
    - models.py
    - state.py
```

### Chapter 5.1 -  Inside the state folder

11. Create models boilderplate in models.py

```bash
# models.py



import reflex as rx


class User(rx.Model, table=True):
    email: str
    password: str


class Contact(rx.Model, table=True):
    user_email: str
    contact_name: str
    email: str
    stage: str = "lead"
```

12. Create state boilderplate in state.py

```bash
# state.py



from typing import Optional
import reflex as rx
from .models import User, Contact


class State(rx.State):
    """The app state."""

    user: Optional[User] = None
```

13. Create login boilerplate in login.py

```bash
# login.py



import reflex as rx
from .models import User
from .state import State


class LoginState(State):
    """State for the login form."""

    email_field: str = ""
    password_field: str = ""

    def log_in(self):
        with rx.session() as sess:
            user = sess.exec(User.select.where(User.email == self.email_field)).first()
            if user and user.password == self.password_field:
                self.user = user
                return rx.redirect("/")
            else:
                return rx.window_alert("Wrong username or password.")

    def sign_up(self):
        with rx.session() as sess:
            user = sess.exec(User.select.where(User.email == self.email_field)).first()
            if user:
                return rx.window_alert(
                    "Looks like you’re already registered! Try logging in instead."
                )
            else:
                sess.expire_on_commit = False  # Make sure the user object is accessible. https://sqlalche.me/e/14/bhk3
                user = User(email=self.email_field, password=self.password_field)
                self.user = user
                sess.add(user)
                sess.commit()
                return rx.redirect("/")

    def log_out(self):
        self.user = None
        return rx.redirect("/")
```

14. Create boilderplate in **init**.py

```bash
# __init__.py



from .state import State
from .login import LoginState 
from . import models
```

### Chapter 6 - components folder and files

> Important folders: components
>
> Important files: crm.py, navbar.py, **init**.py

15. Create components folder with files:
[
crm.py, navbar.py, **init**.py
  ].
Matching below

```bash
> appname 
  > state
   ...
  > components
    - __init__.py
    - crm.py
    - navbar.py
```

### Chapter 6.1 -  Inside the components folder

16. Create boilderplate in navbar.py

```bash
# navbar.py



from appname.state import State, LoginState
import reflex as rx


def navbar():
    return rx.box(
        rx.hstack(
            rx.link("Pyneknown", href="/", font_weight="medium"),
            rx.hstack(
                rx.cond(
                    State.user,
                    rx.hstack(
                        rx.link(
                            "Log out",
                            color="blue.600",
                            on_click=LoginState.log_out,
                        ),
                        rx.avatar(name=State.user.email, size="md"),
                        spacing="1rem",
                    ),
                    rx.box(),
                )
            ),
            justify_content="space-between",
        ),
        width="100%",
        padding="1rem",
        margin_bottom="2rem",
        border_bottom="1px solid black",
    )
```

17. Create boilderplate in crm.py

```bash
# crm.py



from appname.state import State
from appname.state.models import Contact
import reflex as rx


class CRMState(State):
    contacts: list[Contact] = []
    query = ""

    def get_contacts(self) -> list[Contact]:
        with rx.session() as sess:
            if self.query != "":
                print("Query...")
                self.contacts = (
                    sess.query(Contact)
                    .filter(Contact.user_email == self.user.email)
                    .filter(Contact.contact_name.contains(self.query))
                    .all()
                )
                return
            print("All...")
            self.contacts = (
                sess.query(Contact).filter(Contact.user_email == self.user.email).all()
            )

    def filter(self, query):
        self.query = query
        print("Returning...")
        return self.get_contacts()

    @rx.var
    def num_contacts(self):
        return len(self.contacts)


class AddModalState(CRMState):
    show: bool = False
    name: str = ""
    email: str = ""

    def toggle(self):
        self.show = not self.show

    def add_contact(self):
        if not self.user:
            raise ValueError("No user logged in")
        with rx.session() as sess:
            sess.expire_on_commit = False
            sess.add(
                Contact(
                    user_email=self.user.email, contact_name=self.name, email=self.email
                )
            )
            sess.commit()
            self.toggle()
            return self.get_contacts()


def add_modal():
    return rx.modal(
        rx.modal_overlay(
            rx.modal_content(
                rx.modal_header("Add"),
                rx.modal_body(
                    rx.input(
                        on_change=AddModalState.set_name,
                        placeholder="Name",
                        margin_bottom="0.5rem",
                    ),
                    rx.input(on_change=AddModalState.set_email, placeholder="Email"),
                    padding_y=0,
                ),
                rx.modal_footer(
                    rx.button("Close", on_click=AddModalState.toggle),
                    rx.button(
                        "Add", on_click=AddModalState.add_contact, margin_left="0.5rem"
                    ),
                ),
            )
        ),
        is_open=AddModalState.show,
    )


def contact_row(
    contact,
):
    return rx.tr(
        rx.td(contact.contact_name),
        rx.td(contact.email),
        rx.td(rx.badge(contact.stage)),
    )


def crm():
    return rx.box(
        rx.button("Refresh", on_click=CRMState.get_contacts),
        rx.hstack(
            rx.heading("Contacts"),
            rx.button("Add", on_click=AddModalState.toggle),
            justify_content="space-between",
            align_items="flex-start",
            margin_bottom="1rem",
        ),
        rx.responsive_grid(
            rx.box(
                rx.stat(
                    rx.stat_label("Contacts"), rx.stat_number(CRMState.num_contacts)
                ),
                border="1px solid #eaeaef",
                padding="1rem",
                border_radius=8,
            ),
            columns=["5"],
            margin_bottom="1rem",
        ),
        add_modal(),
        rx.input(placeholder="Filter by name...", on_change=CRMState.filter),
        rx.table_container(
            rx.table(rx.tbody(rx.foreach(CRMState.contacts, contact_row))),
            margin_top="1rem",
        ),
        width="100%",
        max_width="960px",
        padding_x="0.5rem",
    )
```

18. Create boilderplate in **init**.py

```bash
# __init__.py



from .crm import crm
from .navbar import navbar
```

### Chapter 7 - Pages folder and files

> Important folders: pages
>
> Important files: index.py, login.py, **init**.py

19. Create pages folder with files:
[
index.py, login.py, **init**.py
  ].
Matching below

```bash
> appname 
  > state
   ...
  >components
   ...
  > pages
    - __init__.py
    - index.py
    - login.py
```

### Chapter 7.1 -  Inside the pages folder

20. Create boilderplate in index.py

```bash
# index.py



from appname.components import navbar
from appname.components import crm
from appname.state import State
import reflex as rx


def index():
    return rx.vstack(
        navbar(),
        rx.cond(
            State.user,
            crm(),
            rx.vstack(
                rx.heading("Welcome to Pyneknown!"),
                rx.text(
                    "This Reflex example demonstrates how to build a fully-fledged customer relationship management (CRM) interface."
                ),
                rx.link(
                    rx.button(
                        "Log in to get started", color_scheme="blue", underline="none"
                    ),
                    href="/login",
                ),
                max_width="500px",
                text_align="center",
                spacing="1rem",
            ),
        ),
        spacing="1.5rem",
    )
```

21. Create boilderplate in login.py

```bash
# login.py



from appname.state import LoginState
from appname.components import navbar
import reflex as rx


def login():
    return rx.vstack(
        navbar(),
        rx.box(
            rx.heading("Log in", margin_bottom="1rem"),
            rx.input(
                type_="email",
                placeholder="Email",
                margin_bottom="1rem",
                on_change=LoginState.set_email_field,
            ),
            rx.input(
                type_="password",
                placeholder="Password",
                margin_bottom="1rem",
                on_change=LoginState.set_password_field,
            ),
            rx.button("Log in", on_click=LoginState.log_in),
            rx.box(
                rx.link(
                    "Or sign up with this email and password",
                    href="#",
                    on_click=LoginState.sign_up,
                ),
                margin_top="0.5rem",
            ),
            max_width="350px",
            flex_direction="column",
        ),
    )
```

22. Create boilderplate in **init**.py

```bash
# __init__.py



from .index import index
from .login import login
```

### Chapter 8 -  Inside appname.py

> Important files: appname.py

23. Make changes in appname.py

```bash
# appname.py



"""Welcome to Reflex! This file outlines the steps to create a basic app."""
import reflex as rx
from appname.pages import index, login
from appname.state import State


# Add state and page to the app.
app = rx.App(state=State)
app.add_page(index)
app.add_page(login)
app.compile()
```

### Chapter 9 - Database initialization

> Important files: rxconfig.py, alembic.ini, reflex.db

24. Make changes in rxconfig.py

```bash
#rxconfig.py



import reflex as rx


config = rx.Config(
    app_name="appname",
    db_url="sqlite:///reflex.db",
    env=rx.Env.DEV,
)
```

25. Initialize new database

> Initialize alembic and create a migration script with the current schema.

```bash
reflex db init
```


