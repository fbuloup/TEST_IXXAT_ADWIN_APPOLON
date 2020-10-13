```c#
using System;
using System.Threading;
using Ixxat.Vci4;
using Ixxat.Vci4.Bal.Can;

namespace Test_IXXAT
{
    class Program
    {
        
        public enum EventType : byte
        {
            APOLLON_EVENT_BEGINSESSION = 0, // session begining event -> ADWin init
            APOLLON_EVENT_ENDSESSION = 1, // session end event -> ADWin closure
            APOLLON_EVENT_BEGINTRIAL = 2, // trial begin event -> ADWin start streaming status (optionnal)
            APOLLON_EVENT_ENDTRIAL = 3, // trial end event -> ADWin stop streaming status (optionnal)
            APOLLON_EVENT_START = 4, // start stimulus event -> ADWin start action immediately after reception
            APOLLON_EVENT_STOP = 5, // stop stimulus event -> ADWin stop/freeze action immediately after reception
            APOLLON_EVENT_RESET = 6, // reset event -> ADwin reset position to initial condition immediately after reception
            APOLLON_EVENT_ACCEL = 0, // reset event -> ADwin reset position to initial condition immediately after reception
            APOLLON_EVENT_SPEED = 1, // reset event -> ADwin reset position to initial condition immediately after reception
            APOLLON_EVENT_DURATION = 2, // reset event -> ADwin reset position to initial condition immediately after reception
        }

        private static ICanMessageWriter messageWriter;
        private static ICanMessage canMessage;

        private static void writeMessage(byte[] message) {
            canMessage.DataLength = (byte)message.Length;// Message length cannot be more than 8 bytes
            for(int i = 0; i < message.Length; i++) {
                canMessage[i] = message[i];
            }
            messageWriter.SendMessage(canMessage);
            Thread.Sleep(10); //Wait 10ms just to be sure can bus transmition is complete
        }
        static void Main(string[] args)
        {
            // All the stuff to initialize CAN controller
            IVciDeviceManager vciDeviceManager = VciServer.Instance().DeviceManager;
            IVciDeviceList vciDeviceList = vciDeviceManager.GetDeviceList();
            System.Collections.IEnumerator deviceEnumerator = vciDeviceList.GetEnumerator();
            deviceEnumerator.MoveNext();
            IVciDevice vciDevice = (IVciDevice)deviceEnumerator.Current;
            Ixxat.Vci4.Bal.IBalObject busAccessLayer = vciDevice.OpenBusAccessLayer();
            ICanControl canControl = (ICanControl)busAccessLayer.OpenSocket(0, typeof(ICanControl));
            
            // Create CAN socket and channel
            ICanSocket canSocket = (ICanSocket)busAccessLayer.OpenSocket(0, typeof(ICanSocket));
            ICanChannel canChannel = (ICanChannel)busAccessLayer.OpenSocket(0, typeof(ICanChannel));
            canChannel.Initialize(1, 1, true); // dont need buffer as there isnt on ADwin side
            canChannel.Activate();
            messageWriter = canChannel.GetMessageWriter();
            canControl.InitLine(CanOperatingModes.Standard, CanBitrate.Cia125KBit); // Dont need extended mode (11 bits is enought)
            canControl.StartLine();

            // Fill CAN message : this part will allways be the same
            canMessage = (ICanMessage)VciServer.Instance().MsgFactory.CreateMsg(typeof(ICanMessage));
            canMessage.Identifier = 1; // Message ID
            canMessage.TimeStamp = 0; // No delay
            canMessage.ExtendedFrameFormat = false; // 11 bits
            canMessage.RemoteTransmissionRequest = false; //
            canMessage.SelfReceptionRequest = false;
            canMessage.FrameType = CanMsgFrameType.Data; // Will allways be a data frame

            // Some values to be transmitted
            float AngularAcceleration = 0.025F;
            float AngularSpeedSaturation = 1.0F;
            float MaxStimDuration = 1000.25F;

            // Start Session : not sure if it's necessary ?
            Program.writeMessage(new byte[]{(byte)EventType.APOLLON_EVENT_BEGINSESSION});

            // 5 trials
            for (uint i = 0; i < 5; ++i)
            {
                // Start trial
                Program.writeMessage(new byte[] { (byte)EventType.APOLLON_EVENT_BEGINTRIAL });

                // Prepare 6 bytes array to send three times :
                // 0x04 0x00 bytes representation of (AngularAcceleration + i)
                // 0x04 0x01 bytes representation of (AngularSpeedSaturation + i)
                // 0x04 0x02 bytes representation of (MaxStimDuration + i)
                // i is added just to see if monster is alive on ADwin side
                byte[] bytes = new byte[1 + 1 + 4];
                System.Buffer.BlockCopy(new byte[] { (byte)EventType.APOLLON_EVENT_START }, 0, bytes, 0, 1);// Allways 0x04

                // Convert AngularAcceleration to its bytes representation and send
                System.Buffer.BlockCopy(new byte[] { (byte)EventType.APOLLON_EVENT_ACCEL }, 0, bytes, 1, 1);// 0x00 : it's accel
                byte[] bytesValues = BitConverter.GetBytes(BitConverter.SingleToInt32Bits(AngularAcceleration + i)); 
                System.Buffer.BlockCopy(bytesValues, 0, bytes, 2, 4);
                Program.writeMessage(bytes);

                // Convert AngularSpeedSaturation + i to its bytes representation and send
                System.Buffer.BlockCopy(new byte[] { (byte)EventType.APOLLON_EVENT_SPEED }, 0, bytes, 1, 1);// 0x01 : it's speed
                bytesValues = BitConverter.GetBytes(BitConverter.SingleToInt32Bits(AngularSpeedSaturation + i)); 
                System.Buffer.BlockCopy(bytesValues, 0, bytes, 2, 4);
                Program.writeMessage(bytes);

                // Convert MaxStimDuration + i to its bytes representation and send
                System.Buffer.BlockCopy(new byte[] { (byte)EventType.APOLLON_EVENT_DURATION }, 0, bytes, 1, 1);// 0x03 : it's duration
                bytesValues = BitConverter.GetBytes(BitConverter.SingleToInt32Bits(MaxStimDuration + i));
                System.Buffer.BlockCopy(bytesValues, 0, bytes, 2, 4);
                Program.writeMessage(bytes);

                Thread.Sleep(2000);// Trial duration

                Program.writeMessage(new byte[] { (byte)EventType.APOLLON_EVENT_STOP });

                Program.writeMessage(new byte[] { (byte)EventType.APOLLON_EVENT_RESET });

                Program.writeMessage(new byte[] { (byte)EventType.APOLLON_EVENT_ENDTRIAL });

            }

            Program.writeMessage(new byte[] { (byte)EventType.APOLLON_EVENT_ENDSESSION });

            // Garbage collection
            canControl.StopLine();
            messageWriter.Dispose();
            canChannel.Dispose();
            canSocket.Dispose();
            canControl.Dispose();
            busAccessLayer.Dispose();
            vciDevice.Dispose();
            vciDeviceList.Dispose();
            vciDeviceManager.Dispose();
            
        }
    }
}
