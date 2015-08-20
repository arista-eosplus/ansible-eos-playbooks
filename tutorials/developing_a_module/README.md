# Add Configuration for clock timezone

This would fit well under eos_system
Currently just hostname and iprouting

# Task Overview

* pyeapi: Create methods to get/set timezone config
* pyeapi: Create pyeapi system tests
* pyeapi: Create pyeapi unit tests
* ansible-eos: Create timezone key in eos_system
* ansible-eos: Create test cases  

# Tasks
## First we need to get the information from pyeapi
1. Make sure we have pyeapi source locally
  ```
  git clone https://github.com/arista-eosplus/pyeapi.git
  ```
2. Go to pyeapi and create a new feature branch
  ```
  cd pyeapi
  git checkout -b feature-time_zone
  ```
3. Open up pyeapi/api/system.py
  ```
  vi pyeapi/api/system.py
  ```

4. Create a parse function that will get the current timezone
  ```
  def _parse_timezone(self):
      """Parses the global config and returns the assigned timezone

      Returns:
          dict: The configure value for clock timezone.  The returned dict
              object is intended to be merged into the resource dict
      """
      match = re.search(r'^clock timezone ([^\s]+)$', self.config, re.M)
      value = match.group(1) if match else 'Unknown'

      return dict(timezone=value)
  ```

## Then we need a method to set the desired value

1. Create a set function
  ```
  def set_timezone(self, value=None, default=False):
      """Configures the system clock timezone

      EosVersion:
          4.13.7M

      Args:
          value(str): One of the acceptable timezone values

          default (bool): Controls the use of the default keyword

      Returns:
          bool: True if the commands completed successfully otherwise False
      """
      cmd = self.command_builder('clock timezone', value=value, default=default)
      return self.configure(cmd)
  ```

2. Add our parse function to the API get() function, update documentation
  ```
  def get(self):
      """Returns the system configuration abstraction

      The System resource returns the following:

          * hostname (str): The hostname value
          * iprouting (bool): Whether ip routing is enabled
          * timezone (str): The clock timezone value

      Returns:
          dict: Represents the node's system configuration
      """
      resource = dict()
      resource.update(self._parse_hostname())
      resource.update(self._parse_iprouting())
      resource.update(self._parse_timezone())
      return resource
  ```

## Now some pyeAPI Tests
1. First, write some system tests. Open file
  ```
  pyeapi/test/system/test_api_system.py
  ```
2. Update expected keys
  ```
  def test_get(self):
      for dut in self.duts:
          dut.config('default hostname')
          resp = dut.api('system').get()
          keys = ['hostname', 'iprouting', 'timezone']
          self.assertEqual(sorted(keys), sorted(resp.keys()))
  ```
3. Add additional tests
  ```
  def test_get_timezone_with_slash(self):
      for dut in self.duts:
          dut.config('clock timezone US/Eastern')
          response = dut.api('system').get()
          self.assertEqual(response['timezone'], 'US/Eastern')

  def test_get_timezone_with_plus(self):
      for dut in self.duts:
          dut.config('clock timezone Etc/GMT+5')
          response = dut.api('system').get()
          self.assertEqual(response['timezone'], 'Etc/GMT+5')

  def test_get_timezone_with_plus_and_0(self):
      for dut in self.duts:
          dut.config('clock timezone Etc/GMT+0')
          response = dut.api('system').get()
          self.assertEqual(response['timezone'], 'Etc/GMT+0')

  def test_get_timezone_(self):
      for dut in self.duts:
          dut.config('clock timezone Factory')
          response = dut.api('system').get()
          self.assertEqual(response['timezone'], 'Factory')

  def test_set_timezone_with_value(self):
      for dut in self.duts:
          dut.config('default hostname')
          value = 'US/Eastern'
          response = dut.api('system').set_timezone(value)
          self.assertTrue(response, 'dut=%s' % dut)
          value = 'clock timezone %s' % value
          self.assertIn(value, dut.running_config)

  def test_set_timezone_with_no_value(self):
      for dut in self.duts:
          dut.config('default hostname')
          response = dut.api('system').set_timezone()
          self.assertTrue(response, 'dut=%s' % dut)
          value = 'clock timezone UTC'
          self.assertIn(value, dut.running_config)

  def test_set_timezone_with_default(self):
      for dut in self.duts:
          dut.config('clock timezone US/Eastern')
          response = dut.api('system').set_timezone(default=True)
          self.assertTrue(response, 'dut=%s' % dut)
          value = 'clock timezone UTC'
          self.assertIn(value, dut.running_config)

  def test_set_timezone_default_over_value(self):
      for dut in self.duts:
          dut.config('clock timezone US/Eastern')
          response = dut.api('system').set_timezone(value='US/Eastern', default=True)
          self.assertTrue(response, 'dut=%s' % dut)
          value = 'clock timezone UTC'
          self.assertIn(value, dut.running_config)
  ```
4. Add unit tests
  ```
  def test_set_timezone(self):
      for state in ['config', 'negate', 'default']:
          if state == 'config':
              cmds = 'clock timezone US/Eastern'
              func = function('set_timezone', True)
          elif state == 'negate':
              cmds = 'no clock timezone'
              func = function('set_timezone', False)
          elif state == 'default':
              cmds = 'default clock timezone'
              func = function('set_timezone', default=True)
          self.eapi_positive_config_test(func, cmds)
  ```

5. Run some real pyeapi commands in a python interpreter
  ```
  >>> import pyeapi
  >>> node = pyeapi.connect_to('veos_test_01')
  >>> system = node.api('system')
  >>> system.get()
  >>> system.set_timezone('US/Eastern')
  >>> system.get()
  ```

## Onto Ansible!!!

1. Make sure you have the local repo
  ```
  git clone https://github.com/arista-eosplus/ansible-eos.git
  ```
2. Create new branch
  ```
  git checkout -b feature-time_zone
  ```
3. Update library/eos_system.py
  ```
  def set_timezone(module):
      """Configures the clock timezone value
      """
      value = module.attributes['timezone']
      module.log('Invoked set_timezone for eos_system with value %s' % value)
      module.node.api('system').set_timezone(value)
  ```
4. Write test cases
  ```
  - name: configure the system clock timezone to UTC
    arguments:
      - { name: timezone, value: UTC }
      - { name: connection, value: $host }
      - { name: debug, value: true }
    changed: false
    setup:
      - no clock timezone

  - name: configure the system clock timezone to US/Eastern
    arguments:
      - { name: timezone, value: US/Eastern }
      - { name: connection, value: $host }
      - { name: debug, value: true }
    setup:
      - no clock timezone

  - name: configure the system clock timezone to GMT+0
    arguments:
      - { name: timezone, value: 'GMT+0' }
      - { name: connection, value: $host }
      - { name: debug, value: true }
    setup:
      - no clock timezone

  - name: configure the system clock timezone to GMT-0
    arguments:
      - { name: timezone, value: 'GMT-0' }
      - { name: connection, value: $host }
      - { name: debug, value: true }
    setup:
      - no clock timezone

  - name: configure the system clock timezone to Etc/GMT-13
    arguments:
      - { name: timezone, value: 'Etc/GMT-13' }
      - { name: connection, value: $host }
      - { name: debug, value: true }
    setup:
      - no clock timezone
  ```
