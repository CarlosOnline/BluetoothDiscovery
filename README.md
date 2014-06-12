BluetoothDiscovery
==================

Discover Bluetooth Devices from c# using registry

Fast & easy code to discover blue tooth devices & their COM Ports on Windows 7 & Windows 8.

Much faster than using the Powershell commands.

```
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.Win32;

namespace BlueToothDevices
{
    /// Using the class
    class Program
    {
        static void Main(string[] args)
        {
            Bluetooth.VerboseMode = args.Length > 0 && args[0] == "true";
            foreach (var device in Bluetooth.GetBluetoothDevices())
            {
                Console.WriteLine("{0}", device);
            }

            Bluetooth.VerboseMode = false;
            foreach (var port in Bluetooth.DeviceNames)
            {
                Console.WriteLine("Device: {0}", port);
            }
        }
    }
    
    /// The Bluetooth discovery code
    
    public class BluetoothDevice
    {
        public string Name { get; set; }
        public string Port { get; set; }
        public string DeviceId { get; set; }

        public override string ToString()
        {
            return string.Format("{0} [ {1} ] {2}", Name, Port, DeviceId);
        }
    }

    public class Bluetooth
    {
        private const string NullDeviceId = "000000000000";
        private const string BluetoothRegKey = @"SYSTEM\CurrentControlSet\Enum\BTHENUM";
        private const string BluetoothRegKey7 = @"SYSTEM\CurrentControlSet\services\BTHPORT\Parameters\LocalServices";
        private readonly Dictionary<string, BluetoothDevice> _devices = new Dictionary<string, BluetoothDevice>();
        public BluetoothDevice[] Devices { get { return _devices.Values.ToArray(); } }

        public static bool VerboseMode { get; set; }
        private static void Verbose(string message, params object[] args)
        {
            if (VerboseMode)
                Console.WriteLine("{0}", string.Format(message, args));
        }

        public static List<string> DeviceNames
        {
            get
            {
                var devices = new List<string>();
                foreach (var device in GetBluetoothDevices())
                {
                    if (string.IsNullOrEmpty(device.Port))
                        continue;

                    devices.Add(string.Format("{0} {1}", device.Name, device.Port));
                }
                return devices;
            }
        }

        public static BluetoothDevice[] GetBluetoothDevices()
        {
            var bluetooth = new Bluetooth();
            bluetooth.GetDeviceNames();
            bluetooth.SetDevicePorts();
            return bluetooth.Devices;
        }

        private void GetDeviceNamesForWindows7()
        {
            using (var key = Registry.LocalMachine.OpenSubKey(BluetoothRegKey7, false))
            {
                if (key == null)
                {
                    Console.WriteLine("GetLocalServices: missing HKLM {0}", BluetoothRegKey7);
                    return;
                }

                foreach (var name in key.GetSubKeyNames())
                {
                    using (var k1 = OpenSubKey(key, name))
                    {
                        if (k1 == null)
                            continue;

                        foreach (var pnpName in k1.GetSubKeyNames())
                        {
                            using (var k2 = OpenSubKey(k1, pnpName))
                            {
                                if (k2 == null)
                                    continue;

                                var friendlyName = GetValue(k2, "ServiceName");
                                var deviceId = GetDeviceToken(GetValue(k2, "AssocBdAddr"));
                                if (string.IsNullOrEmpty(deviceId))
                                {
                                    Console.WriteLine("GetLocalServices: Missing required deviceId");
                                    continue;
                                }

                                var deviceInfo = new BluetoothDevice
                                {
                                    Name = friendlyName,
                                    DeviceId = deviceId,
                                };
                                Verbose("GetLocalServices: Add {0}", deviceInfo);
                                _devices.Add(deviceId, deviceInfo);
                            }
                        }
                    }
                }
            }
        }

        private void GetDeviceNames()
        {
            GetDeviceNamesForWindows7();
            GetDeviceNamesForWindows8();
        }

        private void GetDeviceNamesForWindows8()
        {
            using (var key = Registry.LocalMachine.OpenSubKey(BluetoothRegKey, false))
            {
                if (key == null)
                {
                    Console.WriteLine("GetDeviceNamesForWindows8: missing HKLM {0}", BluetoothRegKey);
                    return;
                }

                foreach (var name in key.GetSubKeyNames())
                {
                    using (var k1 = OpenSubKey(key, name))
                    {
                        if (k1 == null)
                            continue;

                        foreach (var pnpName in k1.GetSubKeyNames())
                        {
                            using (var pnpKey = OpenSubKey(k1, pnpName))
                            {
                                if (pnpKey == null)
                                    continue;

                                BluetoothDevice deviceInfo;
                                var deviceId = GetDeviceName2(pnpName);
                                if (string.IsNullOrEmpty(deviceId))
                                    continue;

                                var friendlyName = GetValue(pnpKey, "FriendlyName", pnpName);
                                if (!_devices.TryGetValue(deviceId, out deviceInfo))
                                {
                                    deviceInfo = new BluetoothDevice();
                                    _devices.Add(deviceId, deviceInfo);
                                }

                                if (string.IsNullOrEmpty(deviceInfo.Name))
                                    deviceInfo.Name = friendlyName;
                                deviceInfo.DeviceId = deviceId;
                                Verbose("GetDevicenamesForWindows8: Add {0}", deviceInfo);
                            }
                        }
                    }
                }
            }
        }

        private void SetDevicePorts()
        {
            using (var key = Registry.LocalMachine.OpenSubKey(BluetoothRegKey, false))
            {
                if (key == null)
                {
                    Console.WriteLine("SetDevicePorts: missing HKLM {0}", BluetoothRegKey);
                    return;
                }

                foreach (var name in key.GetSubKeyNames())
                {
                    using (var k1 = OpenSubKey(key, name))
                    {
                        if (k1 == null)
                            continue;

                        foreach (var pnpName in k1.GetSubKeyNames())
                        {
                            using (var pnpKey = OpenSubKey(k1, pnpName))
                            {
                                if (pnpKey == null)
                                    continue;

                                BluetoothDevice deviceInfo;
                                var isLocalMfg = name.Contains("LOCALMFG");
                                if (!isLocalMfg)
                                    continue;

                                var deviceId = GetDeviceName(pnpName);
                                if (deviceId == NullDeviceId)
                                    continue;

                                if (string.IsNullOrEmpty(deviceId))
                                {
                                    Console.WriteLine("SetDevicePorts:Missing DeviceId for {0}\\{1}", name, pnpName);
                                    continue;
                                }

                                if (!_devices.TryGetValue(deviceId, out deviceInfo))
                                {
                                    Console.WriteLine("SetDevicePorts: Missing DeviceInfo for device: {0} {1}\\{2}", deviceId, name, pnpName);
                                    continue;
                                }

                                if (!string.IsNullOrEmpty(deviceInfo.Port))
                                    continue;

                                if (!pnpKey.GetSubKeyNames().Contains("Device Parameters"))
                                {
                                    Console.WriteLine("SetDevicePorts:Missing Device Parameters for {0}\\{1}", name, pnpName);
                                    continue;
                                }

                                var portName = GetPortName(pnpKey);
                                if (string.IsNullOrEmpty(portName))
                                {
                                    Console.WriteLine("SetDevicePorts: Missing PortName for {0}", pnpName);
                                    continue;
                                }
                                deviceInfo.Port = portName;
                            }
                        }
                    }
                }
            }
        }

        public static string GetDeviceGuid(string value)
        {
            Verbose("GetDeviceGuid: {0}", value);

            var idx = value.IndexOf("}");
            if (idx > 0)
            {
                var id = value.Substring(0, idx);
                if (id.Length == 39)
                {
                    Verbose("GetDeviceGuid: returns {0}", id);
                    return id;
                }
            }
            return null;
        }

        private static string GetDeviceName(string value)
        {
            //Verbose("GetDeviceName: {0}", value);

            var parts1 = value.Split('_');
            if (parts1.Length <= 1)
            {
                Console.WriteLine("GetDeviceName: name parse error on {0}", value);
                return "";
            }

            var parts2 = parts1[0].Split('&');
            if (parts2.Length == 0)
            {
                Console.WriteLine("GetDeviceName: name parse error on {0}", value);
                return "";
            }
            var id = parts2[parts2.Length - 1];

            Verbose("GetDeviceName: returns {0}", id);
            return id;
        }

        private static string GetDeviceName2(string value)
        {
            //Verbose("GetDevicename2: {0}", value);

            var parts = value.Split('_');
            if (parts.Length <= 1)
            {
                Console.WriteLine("GetDeviceName2: name parse error on {0}", value);
                return "";
            }
            var deviceId = parts[1];
            if (string.IsNullOrEmpty(deviceId))
            {
                Console.WriteLine("GetDeviceName2: parse error on {0}", value);
                return "";
            }
            Verbose("GetDeviceName2: returns {0}", deviceId);
            return deviceId;
        }

        private static string GetPortName(RegistryKey key)
        {
            using (var paramsKey = OpenSubKey(key, "Device Parameters"))
            {
                if (paramsKey == null || !paramsKey.GetValueNames().Contains("PortName"))
                    return null;

                var portName = (string)paramsKey.GetValue("PortName");
                Verbose("GetPortName: {0}", portName);
                return portName;
            }
        }

        /// <summary>
        /// "AssocBdAddr"=hex:8f,68,c1,6c,6e,64,00,00 to 646E6CC1688F
        /// </summary>
        /// <param name="value"></param>
        /// <returns></returns>
        public static string GetDeviceToken(string value)
        {
            if (string.IsNullOrEmpty(value))
                return null;

            var parts = value.ToUpper().Split('-');
            if (parts.Length < 6)
                return null;

            var result = string.Join("", parts.Take(6).Reverse());
            Verbose("GetDeviceToken: {0}", result);

            return result;
        }

        private static string GetValue(RegistryKey key, string name, string defaultValue = null)
        {
            if (!key.GetValueNames().Contains(name))
            {
                Console.WriteLine("Missing Value: {0}", name);
                return defaultValue;
            }
            var value = key.GetValue(name);
            var result = value as string;
            if (result != null)
            {
                Verbose("GetValue: {0} {1}", name, result);
                return result;
            }
            var bytes = value as byte[];
            if (bytes != null)
            {
                result = BitConverter.ToString(bytes);
                Verbose("GetValue: {0} {1}", name, result);
                return result;
            }

            return null;
        }

        private static RegistryKey OpenSubKey(RegistryKey key, string name)
        {
            if (key == null)
                return null;

            var parent = key.Name.Replace(@"HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\", "");
            Verbose("OpenSubKey: {0}\\{1}", parent, name);

            var result = key.OpenSubKey(name, false);
            if (result == null)
                Console.WriteLine("OpenSubKey: {0}\\{1}", parent, name);
            return result;
        }

    }
}

```
