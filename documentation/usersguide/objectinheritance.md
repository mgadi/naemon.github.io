---
layout: doctoc
title: Object Inheritance
---
<span class="glyphicon glyphicon-arrow-right"></span> See Also: <a href="objectdefinitions.html">Object Configuration</a>, <a href="objecttricks.html">Object Tricks</a>, <a href="customobjectvars.html">Custom Object Variables</a>, <a href="faststartup.html">Fast Startup Options</a>

### Introduction

This documentation attempts to explain object inheritance and how it can be used in your <a href="objectdefinitions.html">object definitions</a>.

If you are confused about how recursion and inheritance work after reading this, take a look at the sample object config files provided in the Naemon distribution.  If that still doesn't help, drop an email message with a <i>detailed</i> description of your problem to the <i>nagios-users</i> mailing list.

### Basics

There are three variables affecting recursion and inheritance that are present in all object definitions.  They are indicated in red as follows...

<pre>
	define <i>someobjecttype</i>{
		<i>object-specific variables</i> ...
		<font color="red">name		<i>template_name</i></font>
		<font color="red">use		<i>name_of_template_to_use</i></font>
		<font color="red">register	[0/1]</font>
		}
</pre>

The first variable is <i>name</i>.  Its just a "template" name that can be referenced in other object definitions so they can inherit the objects properties/variables.  Template names must be unique amongst objects of the same type, so you can't have two or more host definitions that have "hosttemplate" as their template name.

The second variable is <i>use</i>.  This is where you specify the name of the template object that you want to inherit properties/variables from.  The name you specify for this variable must be defined as another object's template named (using the <i>name</i> variable).

The third variable is <i>register</i>.  This variable is used to indicate whether or not the object definition should be "registered" with Naemon.  By default, all object definitions are registered.  If you are using a partial object definition as a template, you would want to prevent it from being registered (an example of this is provided later).  Values are as follows: 0 = do NOT register object definition, 1 = register object definition (this is the default).  This variable is NOT inherited; every (partial) object definition used as a template must explicitly set the <i>register</i> directive to be <i>0</i>.  This prevents the need to override an inherited <i>register</i> directive with a value of <i>1</i> for every object that should be registered.

### Local Variables vs. Inherited Variables

One important thing to understand with inheritance is that "local" object variables always take precedence over variables defined in the template object.  Take a look at the following example of two host definitions (not all required variables have been supplied):

<pre>
	define host{
		host_name		bighost1
		check_command		check-host-alive
		notification_options	d,u,r
		max_check_attempts	5
		<font color="red">name			hosttemplate1</font>
		}

	define host{
		host_name		bighost2
		max_check_attempts	3
		<font color="red">use			hosttemplate1</font>
		}
</pre>

You'll note that the definition for host <i>bighost1</i> has been defined as having <i>hosttemplate1</i> as its template name.  The definition for host <i>bighost2</i> is using the definition of <i>bighost1</i> as its template object.  Once Naemon processes this data, the resulting definition of host <i>bighost2</i> would be equivalent to this definition:

<pre>
	define host{
		host_name		bighost2
		check_command		check-host-alive
		notification_options	d,u,r
		max_check_attempts	3
		}
</pre>

You can see that the <i>check_command</i> and <i>notification_options</i> variables were inherited from the template object (where host <i>bighost1</i> was defined).  However, the <i>host_name</i> and <i>max_check_attempts</i> variables were not inherited from the template object because they were defined locally.  Remember, locally defined variables override variables that would normally be inherited from a template object.  That should be a fairly easy concept to understand.

{{ site.hint }}If you would like local string variables to be appended to inherited string values, you can do so. Read more about how to accomplish this <a href="#add_string">below</a>.{{ site.end }}

### Inheritance Chaining

Objects can inherit properties/variables from multiple levels of template objects.  Take the following example:

<pre>
	define host{
		host_name		bighost1
		check_command		check-host-alive
		notification_options	d,u,r
		max_check_attempts	5
		<font color="red">name			hosttemplate1</font>
		}

	define host{
		host_name		bighost2
		max_check_attempts	3
		<font color="red">use			hosttemplate1</font>
		<font color="red">name			hosttemplate2</font>
		}

	define host{
		host_name		bighost3
		<font color="red">use			hosttemplate2</font>
		}
</pre>

You'll notice that the definition of host <i>bighost3</i> inherits variables from the definition of host <i>bighost2</i>, which in turn inherits variables from the definition of host <i>bighost1</i>.  Once Naemon processes this configuration data, the resulting host definitions are equivalent to the following:

<pre>
	define host{
		host_name		bighost1
		check_command		check-host-alive
		notification_options	d,u,r
		max_check_attempts	5
		}

	define host{
		host_name		bighost2
		check_command		check-host-alive
		notification_options	d,u,r
		max_check_attempts	3
		}

	define host{
		host_name		bighost3
		check_command		check-host-alive
		notification_options	d,u,r
		max_check_attempts	3
		}
</pre>

There is no inherent limit on how "deep" inheritance can go, but you'll probably want to limit yourself to at most a few levels in order to maintain sanity.

### Using Incomplete Object Definitions as Templates

It is possible to use incomplete object definitions as templates for use by other object definitions.  By "incomplete" definition, I mean that all required variables in the object have not been supplied in the object definition.  It may sound odd to use incomplete definitions as templates, but it is in fact recommended that you use them.  Why?  Well, they can serve as a set of defaults for use in all other object definitions.  Take the following example:

<pre>
	define host{
		check_command		check-host-alive
		notification_options	d,u,r
		max_check_attempts	5
		<font color="red">name			generichosttemplate</font>
		<font color="red">register			0</font>
		}

	define host{
		host_name		bighost1
		address			192.168.1.3
		<font color="red">use			generichosttemplate</font>
		}

	define host{
		host_name		bighost2
		address			192.168.1.4
		<font color="red">use			generichosttemplate</font>
		}
</pre>

Notice that the first host definition is incomplete because it is missing the required <i>host_name</i> variable.  We don't need to supply a host name because we just want to use this definition as a generic host template.  In order to prevent this definition from being registered with Naemon as a normal host, we set the <i>register</i> variable to 0.

The definitions of hosts <i>bighost1</i> and <i>bighost2</i> inherit their values from the generic host definition.  The only variable we've chosed to override is the <i>address</i> variable.  This means that both hosts will have the exact same properties, except for their <i>host_name</i> and <i>address</i> variables.  Once Naemon processes the config data in the example, the resulting host definitions would be equivalent to specifying the following:

<pre>
	define host{
		host_name		bighost1
		address			192.168.1.3
		check_command		check-host-alive
		notification_options	d,u,r
		max_check_attempts	5
		}

	define host{
		host_name		bighost2
		address			192.168.1.4
		check_command		check-host-alive
		notification_options	d,u,r
		max_check_attempts	5
		}
</pre>

At the very least, using a template definition for default variables will save you a lot of typing.  It'll also save you a lot of headaches later if you want to change the default values of variables for a large number of hosts.

### Custom Object Variables

Any <a href="customobjectvars.html">custom object variables</a> that you define in your host, service, or contact definition templates will be inherited just like other standard variables.  Take the following example:

<pre>
	define host{
		_customvar1		somevalue  ; <-- Custom host variable
		_snmp_community		public	; <-- Custom host variable
		<font color="red">name			generichosttemplate</font>
		<font color="red">register			0</font>
		}

	define host{
		host_name		bighost1
		address			192.168.1.3
		<font color="red">use			generichosttemplate</font>
		}
</pre>

The host <i>bighost1</i> will inherit the custom host variables <i>_customvar1</i> and <i>_snmp_community</i>, as well as their respective values, from the <i>generichosttemplate</i> definition.  The effective result is a definition for <i>bighost1</i> that looks like this:

<pre>
	define host{
		host_name		bighost1
		address			192.168.1.3
		_customvar1		somevalue
		_snmp_community		public
		}
</pre>

<a name="cancel_string"></a>

### Cancelling Inheritance of String Values

In some cases you may not want your host, service, or contact definitions to inherit values of string variables from the templates they reference.  If this is the case, you can specify "<b>null</b>" (without quotes) as the value of the variable that you do not want to inherit.  Take the following example:

<pre>
	define host{
		event_handler		my-event-handler-command
		<font color="red">name			generichosttemplate</font>
		<font color="red">register			0</font>
		}

	define host{
		host_name		bighost1
		address			192.168.1.3
		event_handler	null
		<font color="red">use			generichosttemplate</font>
		}
</pre>

In this case, the host <i>bighost1</i> will not inherit the value of the <i>event_handler</i> variable that is defined in the <i>generichosttemplate</i>.  The resulting effective definition of <i>bighost1</i> is the following:

<pre>
	define host{
		host_name		bighost1
		address			192.168.1.3
		}
</pre>

<a name="add_string"></a>

### Additive Inheritance of String Values

Naemon gives preference to local variables instead of values inherited from templates.  In most cases local variable values override those that are defined in templates.  In some cases it makes sense to allow Naemon to use the values of inherited <i>and</i> local variables together.

This "additive inheritance" can be accomplished by prepending the local variable value with a plus sign (<b>+</b>).  This features is only available for standard (non-custom) variables that contain string values.  Take the following example:

<pre>
	define host{
		hostgroups		all-servers
		<font color="red">name			generichosttemplate</font>
		<font color="red">register			0</font>
		}

	define host{
		host_name			linuxserver1
		hostgroups		+linux-servers,web-servers
		<font color="red">use			generichosttemplate</font>
		}
</pre>

In this case, the host <i>linuxserver1</i> will append the value of its local <i>hostgroups</i> variable to that from <i>generichosttemplate</i>.  The resulting effective definition of <i>linuxserver1</i> is the following:

<pre>
	define host{
		host_name			linuxserver1
		hostgroups		all-servers,linux-servers,web-servers
		}
</pre>

<a name="implied_inheritance"></a>

### Implied Inheritance

Normally you have to either explicitly specify the value of a required variable in an object definition or inherit it from a template.  There are a few exceptions to this rule, where Naemon will assume that you want to use a value that instead comes from a related object.  For example, the values of some service variables will be copied from the  host the service is associated with if you don't otherwise specify them.

The following table lists the object variables that will be implicitly inherited from related objects if you don't explicitly specify their value in your object definition or inherit them from a template.

<table border="1">
<tr><th>Object Type</th><th>Object Variable</th><th>Implied Source</th></tr>
<tr>
<td rowspan="3"><b>Services</b></td>
<td><i>contact_groups</i></td>
<td><i>contact_groups</i> in the associated host definition</td>
</tr>
<tr>
<td><i>check_period</i></td>
<td><i>check_period</i> in the associated host definition</td>
</tr>
<tr>
<td><i>notification_interval</i></td>
<td><i>notification_interval</i> in the associated host definition</td>
</tr>
<tr>
<td><i>notification_period</i></td>
<td><i>notification_period</i> in the associated host definition</td>
</tr>
<tr>
<td rowspan="3"><b>Host Escalations</b></td>
<td><i>contact_groups</i></td>
<td><i>contact_groups</i> in the associated host definition</td>
</tr>
<tr>
<td><i>notification_interval</i></td>
<td><i>notification_interval</i> in the associated host definition</td>
</tr>
<tr>
<td><i>escalation_period</i></td>
<td><i>notification_period</i> in the associated host definition</td>
</tr>
<tr>
<td rowspan="3"><b>Service Escalations</b></td>
<td><i>contact_groups</i></td>
<td><i>contact_groups</i> in the associated service definition</td>
</tr>
<tr>
<td><i>notification_interval</i></td>
<td><i>notification_interval</i> in the associated service definition</td>
</tr>
<tr>
<td><i>escalation_period</i></td>
<td><i>notification_period</i> in the associated service definition</td>
</tr>
</table>

<a name="impliedescalations"></a>

### Implied/Additive Inheritance in Escalations

Service and host escalation definitions can make use of a special rule that combines the features of implied and additive inheritance.  If escalations 1) do not inherit the values of their <i>contact_groups</i> or <i>contacts</i> directives from another escalation template and 2) their <i>contact_groups</i> or <i>contacts</i> directives begin with a plus sign (+), then the values of their corresponding host or service definition's <i>contact_groups</i> or <i>contacts</i> directives will be used in the additive inheritance logic.

Confused?  Here's an example:

<pre>
define host{
	name 		linux-server
	contact_groups	linux-admins
	...
	}

define hostescalation{
	host_name		linux-server
	contact_groups	+management
	...
	}
</pre>

This is a much simpler equivalent to:

<pre>
define hostescalation{
	host_name		linux-server
	contact_groups	linux-admins,management
	...
	}
</pre>

<a name="importantvalues"></a>

### Important values

Service templates can make use of a special rule which gives precedence to their check_command value. If the check_command is prefixed with an exclamation mark (!), then the template's check_command is marked as important and will be used over the check_command defined for the service (this is styled after CSS syntax, which uses ! as an important attribute).

Why is this useful? It is mainly useful when setting a different check_command for distributed systems. You may want to set a freshness threshold and a check_command that forces the service into a failed state, but this doesn't work with the normal templating system. Using this <i>important</i> flag allows the custom check_command to be written, but a general distributed template can be used to overrule the check_command when used on a central Naemon server.

For instance:

<pre>
# On master
define service {
	name			service-distributed
	register		0
	active_checks_enabled	0
	check_freshness		1
	check_command		<font color="red">!set_to_stale</font>
	}

# On slave
define service {
	name			service-distributed
	register		0
	active_checks_enabled	1
	}

# Service definition, used by master and slave
define service {
	host_name 		host1
	service_description	serviceA
	check_command		check_http...
	use			service-distributed
	...
	}
</pre>

<a name="multiple_templates"></a>

### Multiple Inheritance Sources

Thus far, all examples of inheritance have shown object definitions inheriting variables/values from just a single source.  You are also able to inherit variables/values from multiple sources for more complex configurations, as shown below.

<table border="0">
<tr>
<td>
<pre>
# Generic host template
define host{
	name 			<font color="red">generic-host</font>
	active_checks_enabled	1
	check_interval		10
	...
	register 			0
	}

# Development web server template
define host{
	name 			<font color="green">development-server</font>
	check_interval		15
	notification_options	d,u,r
	...
	register 			0
	}

# Development web server
define host{
	use			<font color="red">generic-host</font>,<font color="green">development-server</font>
	host_name 		devweb1
	...
	}
</pre>
</td>
<td valign="top">
<img src="images/multiple-templates1.png" border="0" alt="Multiple Inheritance Sources" title="Multiple Inheritance Sources" style="padding: 0 0 0 25px;">
</td>
</tr>
</table>

In the example above, <i>devweb1</i> is inheriting variables/values from two sources: <i>generic-host</i> and <i>development-server</i>.  You'll notice that a <i>check_interval</i> variable is defined in both sources.  Since <i>generic-host</i> was the first template specified in <i>devweb1</i>'s <i>use</i> directive, its value for the <i>check_interval</i> variable is inherited by the <i>devweb1</i> host.  After inheritance, the effective definition of <i>devweb1</i> would be as follows:

<pre>
# Development web server
define host{
	host_name 		devweb1
	active_checks_enabled	1
	check_interval		10
	notification_options	d,u,r
	...
	}
</pre>

### Precedence With Multiple Inheritance Sources

When you use multiple inheritance sources, it is important to know how Naemon handles variables that are defined in multiple sources.  In these cases Naemon will use the variable/value from the first source that is specified in the <i>use</i> directive.  Since inheritance sources can themselves inherit variables/values from one or more other sources, it can get tricky to figure out what variable/value pairs take precedence.

<table border="0">
<tr>
<td valign="top">
<p>
Consider the following host definition that references three templates:
</p>
<pre>
# Development web server
define host{
	use 		<font color="red">1</font>, <font color="#FF8040">4</font>, <font color="purple">8</font>
	host_name		devweb1
	...
	}
</pre>
<p>
If some of those referenced templates themselves inherit variables/values from one or more other templates, the precendence rules are shown to the right.
</p>
<p>
Testing, trial, and error will help you better understand exactly how things work in complex inheritance situations like this.  :-)
</p>
</td>
<td valign="top">
<img src="images/multiple-templates2.png" border="0" alt="Multiple Inheritance Sources" title="Multiple Inheritance Sources" style="padding: 0 0 0 25px;">
</td>
</tr>
</table>
