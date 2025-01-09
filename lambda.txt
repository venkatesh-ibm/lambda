import boto3
import json
import subprocess
import tempfile
import os
import ast
from packaging import version
import pip._vendor.pkg_resources as pkg_resources

def get_lambda_functions():
    """Retrieve all Lambda functions in the account."""
    lambda_client = boto3.client('lambda')
    functions = []
    
    paginator = lambda_client.get_paginator('list_functions')
    for page in paginator.paginate():
        functions.extend(page['Functions'])
    
    return functions

def extract_imports_from_file(file_path):
    """Extract import statements from a Python file."""
    try:
        with open(file_path, 'r') as file:
            tree = ast.parse(file.read())
            
        imports = set()
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for name in node.names:
                    imports.add(name.name.split('.')[0])
            elif isinstance(node, ast.ImportFrom):
                if node.module:
                    imports.add(node.module.split('.')[0])
        
        return list(imports)
    except Exception as e:
        print(f"Error parsing file {file_path}: {str(e)}")
        return []

def get_function_dependencies(lambda_function):
    """Get dependencies from both requirements.txt and imports."""
    lambda_client = boto3.client('lambda')
    
    # Get function code
    response = lambda_client.get_function(FunctionName=lambda_function['FunctionName'])
    location = response['Code']['Location']
    
    dependencies = set()
    
    with tempfile.TemporaryDirectory() as tmp_dir:
        # Download and extract function code
        subprocess.run(['wget', '-q', location, '-O', f'{tmp_dir}/function.zip'])
        subprocess.run(['unzip', '-q', f'{tmp_dir}/function.zip', '-d', tmp_dir])
        
        # Check requirements.txt
        req_file = os.path.join(tmp_dir, 'requirements.txt')
        if os.path.exists(req_file):
            with open(req_file, 'r') as f:
                requirements = [line.strip() for line in f if line.strip() 
                              and not line.startswith('#')]
                dependencies.update(requirements)
        
        # Get imports from Python files
        for root, _, files in os.walk(tmp_dir):
            for file in files:
                if file.endswith('.py'):
                    file_path = os.path.join(root, file)
                    imported_modules = extract_imports_from_file(file_path)
                    dependencies.update(imported_modules)
    
    return list(dependencies)

def check_basic_compatibility(dependencies, target_python_version):
    """Basic compatibility check using pip."""
    results = []
    
    for dep in dependencies:
        try:
            # Try to get package name
            try:
                requirement = pkg_resources.Requirement.parse(dep)
                package_name = requirement.name
            except:
                package_name = dep
            
            # Simple pip check
            cmd = f"pip index versions {package_name}"
            result = subprocess.run(cmd.split(), capture_output=True, text=True)
            compatible = f"Programming Language :: Python :: {target_python_version}" in result.stdout
            
            results.append({
                'package': package_name,
                'compatible': compatible
            })
            
        except Exception as e:
            results.append({
                'package': dep,
                'compatible': False,
                'error': str(e)
            })
    
    return results

def main(target_python_version):
    """Generate compatibility report for Lambda functions."""
    functions = get_lambda_functions()
    reports = []
    
    for function in functions:
        function_name = function['FunctionName']
        current_runtime = function['Runtime']
        
        if not current_runtime.startswith('python'):
            continue
        
        print(f"\nAnalyzing: {function_name}")
        print(f"Current runtime: {current_runtime}")
        
        # Get dependencies
        dependencies = get_function_dependencies(function)
        
        report = {
            'function_name': function_name,
            'current_runtime': current_runtime,
            'dependencies_found': dependencies,
            'needs_review': False,
            'notes': []
        }
        
        if not dependencies:
            print("No dependencies found")
            report['notes'].append("No dependencies detected - review function code manually")
            report['needs_review'] = True
        else:
            print(f"Found {len(dependencies)} dependencies")
            compatibility = check_basic_compatibility(dependencies, target_python_version)
            report['compatibility_check'] = compatibility
            
            incompatible = [r for r in compatibility if not r['compatible']]
            if incompatible:
                report['needs_review'] = True
                report['notes'].append(f"Found {len(incompatible)} potentially incompatible dependencies")
        
        reports.append(report)
        print("Status:", "Needs review" if report['needs_review'] else "Likely compatible")
    
    # Save report
    with open('lambda_compatibility_report.json', 'w') as f:
        json.dump(reports, f, indent=2)
    
    return reports

if __name__ == "__main__":
    target_version = "3.9"  # Change this to your target Python version
    reports = main(target_version)
    
    # Print summary
    print("\nSummary:")
    needs_review = [r for r in reports if r['needs_review']]
    print(f"Total functions analyzed: {len(reports)}")
    print(f"Functions needing review: {len(needs_review)}")
