### utils.sh

```bash
#!/bin/bash

# Function to create a directory with a timestamped name
create_timestamped_dir() {
    local base_name="$1"
    local timestamp
    timestamp=$(date +"%Y%m%d_%H%M%S")
    local dir_name="${base_name}_${timestamp}"
    local absolute_path="/tmp/${dir_name}"
    mkdir -p "$absolute_path"
    echo "$absolute_path"
}
```

### setup_project.sh
```bash
#!/bin/bash

# Source the utils.sh file to use its functions
source ./utils.sh

# Create a timestamped directory for the new project
project_dir=$(create_timestamped_dir "my-new-app")
echo "$project_dir"
```