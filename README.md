Here are examples of how to invoke the status-scope-dmr agent in Copilot Chat:

**Creating a new DMR IP plugin:**
@status-scope-dmr Create a new status-scope plugin for the CXL IP that collects link status and reports errors

**Adding a new product to core_analyzer:**

@status-scope-dmr Add Diamond Rapids support to core_analyzer — create the adapter and project plugin

**Debugging a plugin issue:**
@status-scope-dmr The mesh plugin is throwing a KeyError on "mesh_credit_status". Help me debug it

**Understanding classification flow:**
@status-scope-dmr Explain how a core gets classified as "hung_with_mc" and what captures are triggered

**Reviewing an existing plugin:**
@status-scope-dmr Review the pcie ip_plugin and suggest improvements for error detection

**Config registration:**
@status-scope-dmr Register my new upi_plugin in the DMR config.json under the "io" group

**Porting a plugin from another project:**
@status-scope-dmr Port the memory controller plugin from Novalake to DMR — what adapter changes are needed?

**Adding a custom adapter override:**
@status-scope-dmr Override get_ww_device_templates() in the DMR adapter to add FSCP watch window paths

You invoke it by typing `@status-scope-dmr` followed by your request in the Copilot Chat panel. 
The agent will use its knowledge of the three workspace locations (DMR plugins, bigcore core_analyzer, framework package) 
to help with the task.