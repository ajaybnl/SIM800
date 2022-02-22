# Aspen SIM800 Proper Usage Guide & Library Updated to NONBLOCKING MODE

Now it does not waits in loops in the library.
No More TimeOut in the serial.

## For Starters, Check the included PDF file for command names.

### Help documents
* Library Builtin PDF Document
* https://m2msupport.net/m2msupport/atcreg-network-registration/
* https://mt-system.ru/sites/default/files/simcom_sim800_series_bluetooth_application_guide_v1.01.pdf
* https://www.elecrow.com/wiki/images/2/20/SIM800_Series_AT_Command_Manual_V1.09.pdf

















# Command Usage:



## Detect Call ongoing (incoming or outgoing)

```cpp

bool gsmisincall(){
  //is in call?
      SIM.activity(EXE);    
    bool isincall=SIM.reply(": 4");   
    return (isincall);

}
```

## Extended Error Check Function


```cpp
bool iserr(char *errmatch1 = NULL)
{

  if (SIM.reply("ERR"))
  {
    return false;
  }
  else if (SIM.reply(errmatch1) && errmatch1 != NULL)
  {
    return false;
  }
  else if (SIM.reply("OK"))
  {
    return true;
  }
  return true;
}

```





## Initial Test for various things

```cpp
/****************************************************************************************************************************
*****************************************************************************************************************************/


void gsmloop()
{

 
  // every 3 seconds mandatory
  if ((((millis() - gsmtimer) / 1000) > 3) || gsmtimer == 0) {
    gsmtimer = millis();
    bool ret = false;
    //------------------------------------------
  if(gsmtestmode==0){
  
      if (de) debug("Entered Gsm Settings");

      // Initialize AT commands
      ret = gsminit();
      if (ret) {        
         gsmtestmode++;
         isaterr=false;
      }else{
      if(de)debug("!AT Error");
      isaterr = true;
      return;  
      }

      
  }else if(gsmtestmode==1){

  SIM.pinCode(GET);
  debug(SIM.getBuffer());
  if(SIM.reply(": READY")){
    issimerr = false;
  gsmtestmode++;
  }else{
    issimerr = true;
    if(de)debug("!SIM Error");
    led(NETLED,2);
    return;     
  }



  }else if(gsmtestmode==2){
      // Initialize sim800 settings

      ret = gsmsetparams();

      if (ret) {
        gsmtestmode++;
      }else{
       if(de)debug("!SetParams Error");
       led(NETLED,3);
        return;
      }

  }else if(gsmtestmode==3){

      // Get Career
      SIM.netOps(GET);

      if (SIM.reply(",\"")) {
        // Career OK
        isnetworkerr=false;
        gsmtestmode++;
      }else{
      if(de)debug("!Career Not Assigned");
      isnetworkerr=true;      
      led(NETLED,4);
        return;
      }


  }else if(gsmtestmode==4){

      // Test Network Strength (Non Resetting)

      int signal = gsmgetnetworkstrength();
      if (signal >= 10) {
        issignalerr = false;
         gsmtestmode++;
      } else{
        issignalerr = true;
        if (de) debug("Low Signal");
        led(NETLED,5);
          return;
      }   

  }

  if(gsmtestmode==5){
  gsmtestmode++;
  //Call or Sms Me
  
  
  }


  if(gsmtestmode>=5){
  // All Tests Passed.
  gsmsetupdone=true;
  led(NETLED,1);
  }

  }

```



## Handle Messages (Different Kinds) Received


```cpp


  /******************************************************/


  // HandleSms
  if (((millis() - _timer1) / 1000) > 1)
  {
    _timer1 = millis();

    // SIM.test();
    if (SIM.available())
    {

    if (SIM.reply("NO CARRIER")){
    //Call Cut
    isincall = false;

    }else if (SIM.reply("+CPIN")){
        // Sim Status

        if (SIM.reply("NOT READY")){
        //Sim ERR
        
        }else{
        //Sim OK

        }


      }else if (SIM.reply("+CMGS"))
      {
        // Message Sent

      }
      else if (SIM.reply("+CCLK"))
      {
        // Set Clock

        char *data1 = SIM.getBuffer();
        gettimedate(data1);
      }
      else if (SIM.reply("PSUTTZ"))
      {
        // Set Clock
      }
      else if (SIM.reply("+CIEV"))
      {

        // Network

   //      Call Ready
   //    SMS Ready
   //    *PSUTTZ: 2022,2,17,5,56,7,"+22",0    DST: 0
   //    +CIEV: 10,"40402","airtel","airtel", 0, 0
   

      }
      else if (SIM.reply("+CMTI"))
      {
        // Message Received
        detectsmscommand();
      }
      else if (SIM.reply("+CLIP"))
      {
        // Phone Call

        char *buff = SIM.getBuffer();

        if (authsender(buff))
        {
          //Accept Call
          if(de)debug ("Accept Call");
          isincall = true;
          SIM.answerCall();
        }else{

          //Reject Call
          if(de)debug ("Reject Call");
          SIM.gsmBusy(SET,"1");
        }

        // AUTHNO
      }
      else if (SIM.reply("+DTMF"))
      {
        // DTMF Detect

        String m = String(SIM.getBuffer());
        int a1 = m.indexOf("+DTMF");
        m = m.substring(a1+7, m.indexOf(",", a1));
        int code1 = m.toInt();
       if (de){
          debug("Got :" + m);        
          debug("Code:" + String(code1));
      }
        if (code1 == 1)
        {
          // Play On
          SIM.record(SET, "4,\"C:\\User\\1.amr\",0,100,0");
          motoron=true;
          motoronoff(motoron);
        }
        else if (code1 == 0)
        {
          // Play OFF
          SIM.record(SET, "4,\"C:\\User\\2.amr\",0,100,0");
          motoron=true;
          motoronoff(motoron);
        }
        else if (code1 == 9)
        {

          if (motoron)
          {

            // Play On
            SIM.record(SET, "4,\"C:\\User\\1.amr\",0,100,0");
          }
          else
          {

            // Play Off
            SIM.record(SET, "4,\"C:\\User\\2.amr\",0,100,0");
          }
        }
      }
     // SIM.clearBuffer();
      
    }
  }

}

```





## Get Auth Number

```cpp
  /****************************************************************************************************************************
  *****************************************************************************************************************************/

void gsmsetownerno(char *number1)
{
  strcpy(AUTHNO, number1);
}


```




## Initialize SIM800 and test AT 


```cpp
/****************************************************************************************************************************
*****************************************************************************************************************************/



bool gsminit()
{

  // SIM
  SIM.begin(38400);
  delay(100);

  SIM.setTimeout(3000);
  // SIM.cmdBenchmark(true);

  // Test AT Commands
  bool ret = false;

  int trys = 5;

  while (trys >= 1)
  {
    trys = trys - 1;

    SIM.test();
    if (SIM.isError())
    {
      if (de)
        debug("Error in AT Commands");
    }
    else
    {
      if (de)
        debug("AT Test OK");

      ret = true;

      return (ret);
    }
    delay(1000);
  }

  return (ret);
}


/****************************************************************************************************************************
*****************************************************************************************************************************/

```




## Check Network Strength (0-30)


```cpp

/****************************************************************************************************************************
*****************************************************************************************************************************/


int gsmgetnetworkstrength(){
  // Check Signal and Wait for it
  int q = 0;
  bool ret = false;

  SIM.signalQuality(EXE);

  if (SIM.reply("CSQ"))
  {
    ret = iserr(" 0,0");
    if (!ret)
    {
      if (de)
        debug("signalQuality Err");
    }

    String m = String(SIM.getBuffer());
    int a1 = m.indexOf(":") + 2;
    m = m.substring(a1, m.indexOf(",", a1));
    q = m.toInt();

    if (de)
      debug("Signal : " + String(q));
  }

  return q;
}
```










## Send Sms (To Auth User)

```cpp
/****************************************************************************************************************************
*****************************************************************************************************************************/

bool gsmsendsms(char *msg)
{
  char a[] = "\"";
  char c[] = "\"";
  char *b = AUTHNO;
  char *d = "";

  strcat(d, a);
  strcat(d, b);
  strcat(d, c);

  // char msg1[100];
  // msg.toCharArray(msg1, 100);

  SIM.smsFormat(SET, "1");

  SIM.smsSend(d, msg);

  if (SIM.isError())
  {
    if (de)
      debug("SMS ERROR");
    // SIM.clearBuffer();
    return false;
  }
  if (de)
    debug("SMS Sent");
  // SIM.clearBuffer();
  return true;
}
```




## Initiate Call

```cpp
/****************************************************************************************************************************
*****************************************************************************************************************************/

bool gsmcall(String number1)
{

  String data1 = "+91" + number1 + ";";
  char buff[18];
  data1.toCharArray(buff,18);
if(de)debug(data1);

  SIM.originCall(buff);
  if (SIM.isError())
  {
    if (de)
      debug("CALL ERROR");
    // SIM.clearBuffer();
    return false;
  }
  //delay(20000);
  //SIM.endCall();
  // SIM.clearBuffer();
  if (de)
    debug("Call Sent");
  return true;
}
```




## Detect Command from SMS

```cpp
/****************************************************************************************************************************
*****************************************************************************************************************************/


void detectsmscommand()
{
  SIM.clearBuffer();
  // Read Sms
  SIM.smsFormat(SET, "1");
  SIM.smsRead(SET, "1");

  char *buff = SIM.getBuffer();

  if (authsender(buff))
  {
    buff = NULL;
    // Select Command

    //--------------------------- detect message

    SIM.clearBuffer();

    // Delete All SMS
    SIM.smsList(SET, "\"ALL\"");
    if (SIM.reply("REC"))
    {
      if (de)
        debug("SIM HAS MESSAGES : Deleting...");
      SIM.smsDel(SET, "1,4");
      SIM.clearBuffer();
    }
  }
}
```




## Authorize the Call or Sms Sender Number

```cpp


/****************************************************************************************************************************
*****************************************************************************************************************************/

bool authsender(char *buffer1)
{

  String m = String(buffer1);

  int a1 = m.indexOf("\"+") + 4;
  m = m.substring(a1, m.indexOf(",", a1 + 5) - 1);

  char sender[12];
  m.toCharArray(sender, 12);

  if (de)
    debug("--------Sender:");
  if (de)
    debug(sender);

  char *ph = AUTHNO;

  if (strstr(sender, ph) != NULL)
  {

    if (de)
      debug("--AUTH OK");
    return true;
  }
  else
  {

    if (de)
      debug("--UNKNOWN NUMBER");
    return false;
  }
}

```



## Set Initial Configuration (For CallerID , SMS Notification , etc)


```cpp

/****************************************************************************************************************************
*****************************************************************************************************************************/


bool setinitials()
{
  bool ret = false;



// Set Extended Errors  
SIM.deviceError(SET,"2");

  SIM.setCmdEcho("0");
  ret = iserr(" 0,0");
  if (!ret)
  {
    if (de)
      debug("setCmdEcho Err");
   // return (false);
  }


  SIM.timeStamp(SET, "1");

  ret = iserr(" 0,0");
  if (!ret)
  {
    if (de)
      debug("TimeStanmp Err");
  return (false);
  }

  SIM.dtmfDetect(SET, "1,1000,1,0");

  ret = iserr(NULL);
  if (!ret)
  {
    if (de)
      debug("dtmfDetect Err");
  return (false);
  }

  SIM.sideToneGain(SET, "0,16");
  ret = iserr(NULL);
  if (!ret)
  {
    if (de)
      debug("sideTineGain Err");
  return (false);
  }

  SIM.originCallState(SET, "1");
  ret = iserr(NULL);
  if (!ret)
  {
    if (de)
      debug("originCallState Err");
  return (false);
  }

  SIM.smsFormat(SET, "1");
  ret = iserr(NULL);
  if (!ret)
  {
    if (de) debug("smsFormat Err");    
      issimerr = true;
  return (false);
  }

  SIM.smsNotifyNew(SET, "2,1,0,0,0");
  ret = iserr(NULL);
  if (!ret)
  {
    if (de)
      debug("smsNotifyNew Err");
  return (false);
  }

  SIM.showCallerID(SET, "1");
  ret = iserr(NULL);
  if (!ret)
  {
    if (de)
      debug("showCallerID Err");
  return (false);
  }

  SIM.smsStorage(SET, "\"SM\"");
  ret = iserr(NULL);
  if (!ret)
  {
    if (de)
      debug("smsStorage Err");
   return (false);
  }

  SIM.saveActiveCfg();
  ret = iserr(NULL);
  if (!ret)
  {
    if (de)
      debug("saveActiveCfg Err");
    return (false);
  }

  SIM.clock(GET, "");
  ret = iserr(NULL);
  if (!ret)
  {
    if (de)
      debug("Date Err");
  }

  if (SIM.reply("+CCLK"))
  {
    char *data1 = SIM.getBuffer();
    gettimedate(data1);
  }


  SIM.clearBuffer();

  return true;
}

```






```cpp




## Extract Time Date from Gsm Buffer

/****************************************************************************************************************************
*****************************************************************************************************************************/

void gettimedate(char *buffer1)
{

  // Extract phone number from reply
  String m = String(buffer1);

  char m_year[3];
  char m_month[3];
  char m_day[3];
  char m_hour[3];
  char m_minute[3];
  char m_second[3];

  // Serial.println(m);

  m = m.substring(m.indexOf("\"") + 1, m.lastIndexOf("\"") - 3);
  // Serial.println(m);
  m.substring(0, 2).toCharArray(m_year, sizeof(m_year));
  m.substring(3, 5).toCharArray(m_month, sizeof(m_month));
  m.substring(6, 8).toCharArray(m_day, sizeof(m_day));
  m.substring(9, 11).toCharArray(m_hour, sizeof(m_hour));
  m.substring(12, 14).toCharArray(m_minute, sizeof(m_minute));
  m.substring(15, 17).toCharArray(m_second, sizeof(m_second));

  myear = atoi(m_year);
  mmonth = atoi(m_month);
  mday = atoi(m_day);
  mhour = atoi(m_hour);
  mminute = atoi(m_minute);
  msecond = atoi(m_second);

  if (de)
  {
    Serial.print("DateTime: ");
    Serial.print(mday);
    Serial.print('/');
    Serial.print(mmonth);
    Serial.print('/');
    Serial.print(myear);
    Serial.print(" - ");
    Serial.print(mhour);
    Serial.print(':');
    Serial.print(mminute);
    Serial.print(':');
    Serial.print(msecond);
    Serial.println();
  }
  
}
```




## untested

```cpp

/****************************************************************************************************************************
*****************************************************************************************************************************/

bool gsmiscallready() {
  
  SIM.queryCallReady(GET);
  
  if (SIM.reply(": 1")) {
    return (true);
  } else {
    return (false);
  }

}


```
