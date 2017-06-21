---
layout: post
title: "Code with ws-discovery in win32"
lang: en
comments: true
summary: "This article describes the detail about ws-discovery "
tags: [ws-discovery,socket,udp]
---

# WS-Discovery #
**Web Services Dynamic Discovery** [(**WS-Discovery**)](https://en.wikipedia.org/wiki/WS-Discovery)is a technical specification that defines a multicast discovery protocol to locate services on a local network. It operates over TCP and UDP **port 3702 **and uses IP multicast address **239.255.255.250**. As the name suggests, the actual communication between nodes is done using web services standards, notably SOAP-over-UDP.

UDP ports 139, 445, 1124, 3702 TCP ports 139, 445, 3702, 49179, 5357,5358

The primary mode of discovery is a client searching for one or more target services. To find a target service by the type of the target service, a scope in which the target service resides, or both, a client sends a probe message to a multicast group; target services that match the probe send a response directly to the client.

This blog is going to talk about the using ws-discovery to find device.
## DiscoveryDevice ##
Normally, you send data to client and then receive from client to match your device. But may be there is multi-card in your PC. To consider this situation, you should send data muli-times. And you will send with different net card IP, message id and local port. 

    int DiscoveryDevice()
    {
    	WSAData wsa_data; 
    	if(WSAStartup(0x202,&wsa_data) != 0) 
    		return 0;
    
    	char szMsgGUID[1024] = "";
    	struct hostent *hostinfo = gethostbyname(""); //Multi-card
    	int i = 0;
    	while (hostinfo->h_addr_list[i] != NULL)
    	{
    		IN_ADDR inaddr = *(struct in_addr *) hostinfo->h_addr_list[i];
    		{
    			GUID gguid;
    			::UuidCreate(&gguid); // generated GUID
    
    			sprintf(szMsgGUID, "%08lX-%04X-%04X-%02X%02X-%02X%02X%02X%02X%02X%02X",
    				gguid.Data1, gguid.Data2, gguid.Data3,
    				gguid.Data4[0], gguid.Data4[1], gguid.Data4[2], gguid.Data4[3],
    				gguid.Data4[4], gguid.Data4[5], gguid.Data4[6], gguid.Data4[7]);
    		}
    
    		BOOL bReturn = SendXMLData(szMsgGUID,&inaddr);
    		if (!bReturn)
    		{
    			WSACleanup();
    			return 0;
    		}
    
    		int nFindDevice = 0;
    		for (int nSearchCnt = 0; nSearchCnt < 5; nSearchCnt++)
    		{
    			BOOL bReturn = ListenServer(szMsgGUID, &nFindDevice, pCurList, &inaddr);
    			if (nFindDevice)
    			{
    				break;
    			}	
    		}
    
    		if (nFindDevice > 0)
    		{
    			//add device to list...
    		}
    
    		i++;
    	}
    }
## SendXMLData ##
This function is design to send data to client, before sending, you should create a UDP socket and join to IP multicast address group. Notice that the local port is random, but the dest port is **3702** and the IP multicast address must be **239.255.255.250**.
	

    const long cNetPortStart = 52515;
    const long cNetPortCount = 100;
    BOOL SendXMLData(char *szPcUuid,IN_ADDR * addr)
    {
    	CString strDebug;
    	int nResult = FALSE;
    	if (addr == NULL) { return nResult; }
    	SOCKET sSearchSocket = socket(AF_INET, SOCK_DGRAM, 0); //IPPROTO_UDP
    	if (INVALID_SOCKET == sSearchSocket)
    	{
    		strDebug.Format(_T("socket function failed with error: %u\n"), WSAGetLastError());
    		OutputLog(strDebug);
    		return FALSE;
    	}
    	    
    	srand (GetCurrentThreadId());
    	m_nBroadcastPort = cNetPortStart + (rand() % cNetPortCount);//rand

    	struct sockaddr_in Local_addr;
    	memset(&Local_addr, 0, sizeof(sockaddr_in));
    	Local_addr.sin_port = htons(m_nBroadcastPort);
    	Local_addr.sin_family = AF_INET;
    	Local_addr.sin_addr = *(struct in_addr *) addr;
    	int  nLength = sizeof(sockaddr_in);
    	nResult = bind(sSearchSocket, (struct sockaddr*)&Local_addr, sizeof(Local_addr));
    	if (nResult == SOCKET_ERROR)
    	{
    		strDebug.Format(_T("SendXMLData bind fail: %u\n"), WSAGetLastError());
    		OutputLog(strDebug);
    		shutdown(sSearchSocket, 0x01);
    		closesocket(sSearchSocket);
    		return FALSE;
    	}

    	int iOptVal = 64;
    	if (setsockopt(sSearchSocket, IPPROTO_IP, IP_MULTICAST_TTL, (char*)&iOptVal, sizeof(int)) == SOCKET_ERROR)
    	{
    		strDebug.Format(_T("SendXMLData setsockopt fail: %u\n"), WSAGetLastError());
    		OutputLog(strDebug);
    		shutdown(sSearchSocket, 0x01);
    		closesocket(sSearchSocket);
    		return FALSE;
    	}

    	//Assign Destination Address
    	SOCKADDR_IN DestinationAddr;
    	DestinationAddr.sin_family = AF_INET;
    	DestinationAddr.sin_addr.s_addr = inet_addr("239.255.255.250");
    	DestinationAddr.sin_port = htons(3702);
    
    	char szXml[] = "<soap:Envelope xmlns:soap=\"http://www.w3.org/2003/05/soap-envelope\" xmlns:wsa=\"http://schemas.xmlsoap.org/ws/2004/08/addressing\" xmlns:hpd=\"http://www.hp.com/schemas/imaging/con/discovery/2006/09/19\" xmlns:wsd=\"http://schemas.xmlsoap.org/ws/2005/04/discovery\"><soap:Header><wsa:To>urn:schemas-xmlsoap-org:ws:2005:04:discovery</wsa:To><wsa:Action>http://schemas.xmlsoap.org/ws/2005/04/discovery/Probe</wsa:Action><wsa:MessageID>urn:uuid:BCDFA477-F180-4ABA-9766-76F0B42B95F6</wsa:MessageID></soap:Header><soap:Body><wsd:Probe><wsd:Types>wsdp:Device wscn:ScanDeviceType</wsd:Types></wsd:Probe></soap:Body></soap:Envelope>";
    	int nReturn = sizeof(szXml)+1;
    
    	//replace uuid
    	int nUUIDSize = strlen(szPcUuid);
    	memcpy(szXml + 436, szPcUuid, nUUIDSize);
    
    	nResult = sendto(sSearchSocket, szXml, nReturn, 0, (struct sockaddr*)&DestinationAddr, sizeof(DestinationAddr));
    
    	if (nResult < 0 || nResult < nReturn)
    	{
    		strDebug.Format(_T("SendXMLData sendto fail: %u\n"), WSAGetLastError());
    		OutputLog(strDebug);
    		shutdown(sSearchSocket, 0x01);
    		closesocket(sSearchSocket);
    		return FALSE;
    	}
    	shutdown(sSearchSocket, 0x01);
    	closesocket(sSearchSocket);
    	return TRUE;
    }
## ListenServer ##
This function is design to receive data from client, before receiving, you should create a UDP socket and join to IP multicast address group. The IP multicast address must be **239.255.255.250**, and you can receive data from any ip address.

    BOOL ListenServer(char *szPcUuid, int *npFindDevice, Device* pDeviceList, IN_ADDR * addr)
    {
    	if (szPcUuid == NULL || npFindDevice == NULL || pDeviceList == NULL || addr == NULL)
    		return FALSE;
    
    	CString strDebug;
    
    	BOOL bReturn = FALSE;
    	int	 nResult = FALSE;
    	SOCKET	sSearchSocket = INVALID_SOCKET;
    
    	sSearchSocket = socket(AF_INET, SOCK_DGRAM, 0);
    	if (INVALID_SOCKET == sSearchSocket)
    	{
    		strDebug.Format(_T("socket create fail: %u\n"), WSAGetLastError());
    		OutputLog(strDebug);
    		return bReturn;
    	}
    
    	struct sockaddr_in Local_addr;
    	memset(&Local_addr, 0, sizeof(sockaddr_in));
    	Local_addr.sin_port = htons(m_nBroadcastPort);
    	Local_addr.sin_family = AF_INET; 
    	Local_addr.sin_addr.S_un.S_addr = INADDR_ANY;
    
    	int  nLength = sizeof(sockaddr_in);
    	nResult = bind(sSearchSocket, (struct sockaddr*)&Local_addr, sizeof(Local_addr));
    	if (nResult == SOCKET_ERROR)
    	{
    		strDebug.Format(_T("socket bind error: %u\n"), WSAGetLastError());
    		OutputLog(strDebug);
    
    		shutdown(sSearchSocket, 0x01);
    		closesocket(sSearchSocket);
    		return FALSE;
    	}
    
    	struct ip_mreq mreq;
    	// Join the multicast group from which to receive datagrams.
    	mreq.imr_multiaddr.s_addr = inet_addr("239.255.255.250");
    	mreq.imr_interface.s_addr = INADDR_ANY;
    
    	nResult = setsockopt(sSearchSocket, IPPROTO_IP, IP_ADD_MEMBERSHIP, (char *)&mreq, sizeof(mreq));
    	if (SOCKET_ERROR == nResult)
    	{
    		strDebug.Format(_T("setsockopt addmemebership error: %u\n"), WSAGetLastError());
    		OutputLog(strDebug);
    
    		shutdown(sSearchSocket, 0x01);
    		closesocket(sSearchSocket);
    		return FALSE;
    	}
    
    	struct timeval time_out = { 0 };
    	time_out.tv_sec = 300;
    	if (setsockopt(sSearchSocket, SOL_SOCKET, SO_RCVTIMEO, (char *)&time_out, sizeof(time_out)))
    	{
    		strDebug.Format(_T("setsockopt rcvtimeo error: %u\n"), WSAGetLastError());
    		OutputLog(strDebug);
    
    		shutdown(sSearchSocket, 0x01);
    		closesocket(sSearchSocket);
    		return FALSE;
    	}
    
    	int nRecvLength = 0;
    	struct sockaddr_in source_Addr;
    	int nRequstLength = sizeof(source_Addr);
    	int nBufferSize = 2500;
    	char *pszInbuffer = new char[nBufferSize];
    
    	DWORD dwTimeStart = GetTickCount();
    	do
    	{
    		memset(pszInbuffer, 0, sizeof(char)*nBufferSize);
    		nRecvLength = recvfrom(sSearchSocket, pszInbuffer, nBufferSize, 0, (struct sockaddr*) &source_Addr, &nRequstLength);
    
    		if (nRecvLength <= 0)
    			continue;
    
    		//search pszInbuffer and add device to pDeviceList...
    	} while (nRecvLength>0 || GetTickCount() - dwTimeStart < 500);  //wait 0.5 sec
    
    	if (pszInbuffer)
    	{
    		delete[] pszInbuffer;
    		pszInbuffer = NULL;
    	}
    	shutdown(sSearchSocket, 0x01);
    	closesocket(sSearchSocket);
    
    	return TRUE;
    }
# Export log file #
This function is design to export log, and it is unicode. Before using, you should creat a file named "ExportLog.txt" in the path "c:\Dump",  and you can use it the same way as using OutputDebugString.

    void OutputLog(CString strMsg)
    {
    	try
    	{
    #ifdef _DEBUG
    		OutputDebugString(strMsg);
    #endif
    		CString strLogFile = _T("c:\\Dump\\ExportLog.txt");
    		if (PathFileExists(strLogFile))
    		{
    			CStdioFile outFile(strLogFile, CFile::modeNoTruncate |CFile::modeWrite | CFile::typeBinary);
    			CString msLine; 
    			CTime tt = CTime::GetCurrentTime();
    			msLine = tt.Format(_T("[%Y-%B-%d %A, %H:%M:%S] ")) + strMsg;  //2010-June-10 Thursday, 15:58:12
    			outFile.SeekToEnd();
    			outFile.WriteString( msLine );
    			outFile.Close();
    		}
    	}
    	catch(CFileException *fx)
    	{
    		fx->Delete();
    	}
    }

