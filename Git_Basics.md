Lecture Title : Introduction to Docker

Lecture Date : 10th Nov 2025 (Monday)

## Q4. Git - Working with Files & Staging Area

#### Step 0 : Set the git configuration

```bash
git config –global user.email “you@example.com”

# Cross check
git config --global user.email

git config –global user.name “Your Name”

# Cross check
git config --global user.name
```

#### Step 1 : Copy the sample.txt content

```bash
cp sample.txt /app-metrics-docs/metrics-config.yaml

cp sample.txt /app-metrics-docs/README.md
```

#### Step 2 : Initial Commit

```bash
# check repo status
git status

# Stage both files
git add metrics-config.yaml README.md

# Commit both
git commit -m "Initial metrics config and documentation"
```

#### Step 3 : Modify & check diff

```bash
vi metrics-config.yaml

vi README.md

# check diff
git diff
```

#### Step 4 : Stage & Commit

```bash
# Stage all
git add .

# Unstage the README.md
git reset --staged README.md

# Commit the metrics-config.yaml
git commit -m "Added new parameter to metrics config"
```