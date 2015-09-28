# Using packages.config to manage the project dependencies workflow

## Declare your dependencies

Declare project dependencies directly in packages.config

''' xml
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <package id="xunit" version="2.0.0" targetFramework="net45" />
  <package id="xunit.abstractions" version="2.0.0" targetFramework="net45" />
  <package id="xunit.assert" version="2.0.0" targetFramework="net45" />
  <package id="xunit.core" version="2.0.0" targetFramework="net45" />
  <package id="xunit.extensibility.core" version="2.0.0" targetFramework="net45" />
</packages>
'''

## Grab the packages

1. Requires command-line nuget:

	'''
	choco install nuget.commandline
	'''

2. Download all references packages

	'''
	nuget restore <solution_name.sln>
	'''

## Add project references

From the Package Manager Console:

''' 
update-package -reinstall
'''
