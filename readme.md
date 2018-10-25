# Basic logic structure of a module
the basic logic flow when creating a custom module should resemble the structure below. details for each are provided and found, culminated, in the `library/motd.py`. this content comes directly from an Ansible Fest presentation done by Matt Davis (Ansible). If you have a spare hour, i highly recommend you [watch his presentation](https://www.ansible.com/ansible-module-development-101); it does a much better job explaining and is hilarious. this guide can be used as a reference or for skimming the general ideas presented. 
- docstring with DOCUMENTATION,EXAMPLES, and RETURNS
- imports / function definitions
- instatiation of AnsibleModule class with argument_spec dict 
- possible parameters in your module
    - required_if
    - supports_check_mode
- perform idempotentcy checks
- perform desired module changes
- exit 
- catch \_\_name\_\_ for running as script 

# Testing during development
instead of using ansible to run your module, you can test from within pycharm (any modern IDE). since the ansible class is just expecting json input for the parameters, we can just pass them from a file as a nested json object ANSIBLE_MODULE_ARGS. to load the file in pycharm: run -> edit configuration -> place file path in 'parameters' under 'script path'.

```json
> cat args.txt

{ "ANSIBLE_MODULE_ARGS": {
    "message": "hi mom",
    "state": "present",
    "_ansible_check_mode": True
  }
}
```

# Description of logic

## Docstrings
- DOCUMENTATION
    - module name, author, description 
    - parameters and their options
- EXAMPLES
    - examples how to implement this module
- RETURNS
    - information on what is returned when the module exits

```python
RETURNS = '''
motd_path:
    description: path to MOTD file used by module
    returned: success
    type: string
'''
```

## Imports and function def, instantiate AnsibleModule, and set required 
it is not required to import and utilize the AnsibleModule class. the only requirement of a module is that it must accept and output json. you could even use another language (go) but im not sure the reason you'd do that. using the prebuilt class does provide some nifty tools such as exit functions, arg parsing the json stdin, and arg validation. 

```python
import os
from ansible.module_utils.basic import AnsibleModule

motd_file = '/home/mullendrew/motd'

def main():
    module = AnsibleModule(
        argument_spec=dict(
            message=dict(type='str'),
            state=dict(type='str', choices=['present','absent'], default='present')
        ),
        required_if=([('state', 'present', ['message'])]),
        supports_check_mode=True
    )
```

## Perform idempotentcy checks 
its best practice to perform idempotency checks prior and separate from your manipulation (change actions). in the example, the local var `changed` is set as `false`, if any of the idempotency checks show discrepancy from desired state, flip the bool to `true`. 

this allows for clear, readable logic in your code. it also makes it easy to implement check_mode by separating any system changes from the checks performed. finally, this is the var you will pass to exit_json() for ansible to report at the end of the play.

```python
    message = module.params['message']
    requested_state = module.params['state']
    changed = False

    exists = os.path.exists(motd_file)

    if exists:
        if requested_state == 'present':
            with open(motd_file, 'r') as fd:
                current_message = fd.read().strip()

            if current_message != message:
                changed = True
        else:
            changed = True
    else:
        if requested_state == 'present':
            changed = True
```

## Perform desired module changes
if idempotency checks show a discrepancy between current state and desired state **and** check_mode == false, make the planned changes to system!

```python
    # to properly support check mode, do as much as possible
    # without actually making changes
    if changed and not module.check_mode:
        if requested_state == 'present':
            with open(motd_file, 'w') as fd:
                fd.write(module.params['message'])
        else:
            os.remove(motd_file)
```

## Exit
exit the module by passing the local `changed` var value to exit_json(). you can also pass additional values to provide extra info to your users.

```python
module.exit_json(changed=changed, motd_path=motd_file)
```

## Dunder script catch
common design pattern in python to run the main function even if the module is run as a script.
```python
if __name__ == '__main__':
    main()
```