```VBScript
#INCLUDE ADwinGoldII.INC

' Real time loop frequency
DIM FEG AS LONG

LOWINIT:
  
  FEG = 1000
  GLOBALDELAY = 10^9/(FEG*3.3)

  REM ******** Initialize CAN interface
  INIT_CAN(1)' interface num 1
  SET_CAN_REG(1, 6, 0)
  SET_CAN_REG(1, 7, 0)
  fpar_80 = SET_CAN_BAUDRATE(1, 125000)' 125kbit/s
  REM Receive  Message object 1 Message ID 1 Message ID length 11 bits
  EN_RECEIVE(1, 1, 1, 0)
  
EVENT:
  
  IF(READ_MSG(1,1) = 1) THEN
    PAR_1 = CAN_MSG[9] ' nb bytes received
    PAR_2 = CAN_MSG[1] ' Appolon code
    IF(CAN_MSG[1] = 4) THEN  ' If appolon code is START
      PAR_3 = SHIFT_LEFT(CAN_MSG[6], 24) + SHIFT_LEFT(CAN_MSG[5], 16) + SHIFT_LEFT(CAN_MSG[4], 8) + CAN_MSG[3] ' contruct INTEGER representation
      ' Convert to FLOAT
      IF(CAN_MSG[2] = 0) THEN 
        FPAR_2 = Cast_LongToFloat(PAR_3)' If its ACCEL
      ENDIF
      IF(CAN_MSG[2] = 1) THEN        
        FPAR_3 = Cast_LongToFloat(PAR_3)' If its SPEED
      ENDIF
      IF(CAN_MSG[2] = 2) THEN        
        FPAR_4 = Cast_LongToFloat(PAR_3)' If its DURATION
      ENDIF
    ENDIF    
  ENDIF
