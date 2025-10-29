# Supervisor Manager - Overview

## 🚀 What is Supervisor Manager?

Supervisor Manager is a powerful Python tool designed to simplify the deployment and management of Python applications using Supervisor. It provides an easy-to-use interface for generating supervisor configurations, managing application lifecycles, and automating deployment processes.

## ✨ Key Features

- **🎯 Easy Deployment**: Deploy Python applications with a single command
- **📝 Template System**: Pre-built templates for Flask, Streamlit, Django, FastAPI, and more
- **⚙️ Program Management**: Start, stop, restart, and monitor supervisor programs
- **🔧 Auto-generated Scripts**: Automatically creates shell scripts for your applications
- **📊 Log Management**: Automatic log file configuration and rotation
- **🎮 Interactive Mode**: User-friendly interactive interface
- **🔄 Batch Operations**: Deploy multiple applications at once

## 🏗️ Architecture

The Supervisor Manager consists of several key components:

### Core Classes

- **`SupervisorManager`**: Main class for managing supervisor configurations and programs
- **`AppConfig`**: Data class for application configuration
- **`SupervisorTemplates`**: Predefined templates for common application types

### File Structure

```
supervisor_manager/
├── supervisor_manager.py      # Main application logic
├── supervisor_templates.py    # Predefined templates
├── demo_supervisor.py         # Demo and examples
├── example_deployment.py      # Deployment examples
└── SUPERVISOR_MANAGER_README.md # Basic documentation
```

## 🎯 Use Cases

### 1. Web Application Deployment
Deploy Flask, Streamlit, Django, or FastAPI applications with proper process management.

### 2. Background Services
Run Python scripts, data processors, or Telegram bots as managed services.

### 3. Development to Production
Seamlessly move applications from development to production with supervisor.

### 4. Multi-Application Management
Manage multiple Python applications from a single interface.

## 🔧 Supported Application Types

| Type | Description | Default Port | Template |
|------|-------------|--------------|----------|
| **Flask** | Web applications | 5000 | ✅ |
| **Streamlit** | Data science apps | 8501 | ✅ |
| **Django** | Full-stack web apps | 8000 | ✅ |
| **FastAPI** | Modern API applications | 8000 | ✅ |
| **Telegram Bot** | Bot applications | N/A | ✅ |
| **Data Processor** | Background scripts | N/A | ✅ |
| **Generic** | Any Python script | Custom | ✅ |

## 🚀 Quick Start

### Command Line Usage
```bash
# Deploy a new application
python supervisor_manager.py deploy my_app /path/to/app.py --type streamlit --port 8501

# Manage programs
python supervisor_manager.py manage start my_app
python supervisor_manager.py list

# Interactive mode
python supervisor_manager.py interactive
```

### Python API Usage
```python
from supervisor_manager import SupervisorManager
from supervisor_templates import SupervisorTemplates

# Create manager
manager = SupervisorManager()

# Deploy using template
config = SupervisorTemplates.streamlit_app(
    name="my_app",
    app_path="/path/to/app.py",
    port=8501
)
result = manager.deploy_app(config)
```

## 📁 Generated Files

When you deploy an application, Supervisor Manager creates:

```
/home/gws/deployment/
├── {app_name}.sh              # Shell script to run the app
└── {app_name}/                # App directory
    ├── {app_name}.log         # Stdout log
    └── {app_name}_err.log     # Stderr log

/home/gws/supervisor_configs/
└── {app_name}.conf            # Supervisor configuration

/etc/supervisor/conf.d/
└── {app_name}.conf            # Deployed supervisor config
```

## 🔄 Workflow

1. **Configure**: Define your application settings
2. **Generate**: Create supervisor configuration and shell scripts
3. **Deploy**: Copy configuration to supervisor directory
4. **Manage**: Start, stop, restart, and monitor applications
5. **Monitor**: Check logs and application status

## 🎯 Benefits

- **Simplified Deployment**: No need to manually write supervisor configurations
- **Consistent Setup**: Standardized configuration across all applications
- **Easy Management**: Single interface for all supervisor operations
- **Error Prevention**: Validates configurations before deployment
- **Log Management**: Automatic log rotation and management
- **Environment Handling**: Proper virtual environment activation

## 🔗 Next Steps

- **Installation Guide** - Set up Supervisor Manager
- **Usage Examples** - Learn how to use the tool
- **API Reference** - Detailed API documentation
- **Troubleshooting** - Common issues and solutions

