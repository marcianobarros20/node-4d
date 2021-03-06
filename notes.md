# Commands

These are the supported commands of the protocol:
- LOGIN
- PREPARE-STATEMENT
- EXECUTE-STATEMENT
- CLOSE-STATEMENT
- FETCH-RESULT
- BULK-MODIFY
- LOGOUT
- QUIT

# LOGIN

Syntax:
Command-ID SP LOGIN CRLF
USER-NAME-BASE64 SP : SP Base-64-Encoded-User-Name CRLF
USER-PASSWORD-BASE64 SP: SP Base-64-Encoded-User-Password CRLF
REPLY-WITH-BASE64-TEXT: [Y | N ]
VERSION SP : SP Version-String CRLF
OS-NAME SP : SP OS-Name CRLF
OS-VERSION SP : SP OS-Version CRLF
CRLF

Example:
001 LOGIN CRLF
USER-NAME: UserName CRLF
USER-PASSWORD: UserPassword CRLF
REPLY-WITH-BASE64-TEXT: Y
PROTOCOL-VERSION: 0.1a CRLF CRLF

User password may also be a MD5 hash. In this case, use USER-PASSWORD-MD5 instead of USER-PASSWORD.

# LOGOUT

Syntax:
Command-ID LOGOUT CRLF CRLF

Example:
002 LOGOUT CRLF CRLF

Logging out does not close the socket. You can login again to create a new session.

# EXECUTE-STATEMENT

Syntax:
Command-ID SP EXECUTE-STATEMENT CRLF
STATEMENT SP : SP Statement text CRLF
OUTPUT-MODE SP: SP Debug or Release CRLF
PREFERRED-IMAGE-TYPES : A space-separated list of target image formats FIRST-PAGE-SIZE : Number of records returned with response (default is 100) CRLF PARAMETER-TYPES: SP A space-separated list of XToolBox types SP CRLF FULL-ERROR-STACK: SP Y or N SP CRLF
CRLF
*Parameter values in binary mode as described below.*

Example:
003 Execute-Statement
Statement: select * from mytable
Output-Mode: release

Syntax reponse:
Command-ID SP Status(OK or Error) CRLF
STATEMENT-ID SP : SP A server-assigned statement id CRLF
COMMAND-COUNT SP : SP Amount of commands in the statement CRLF
RESULT-TYPE SP : SP Result-Set or Update-Count CRLF
COLUMN-COUNT SP : SP Column count in the result set CRLF
ROW-COUNT SP : SP Row count in the result set CRLF
COLUMN-TYPES SP : SP A space-separated list of XToolBox types CRLF
COLUMN-ALIASES SP : SP Space-separated aliases in optional [ and ] CRLF
COLUMN-UPDATEABILITY SP : SP Space-separated Y or N CRLF
ROW-COUNT-SENT SP : SP An amount of rows sent in this initial response CRLF CRLF

Example response:
003 OK
Statement-ID:2
Command-Count:1
Result-Type:Result-Set
Column-Count:3
Row-Count:16
Column-Types:VK_LONG VK_STRING VK_STRING
Column-Aliases:[EMPNO] [ENAME] [JOB]
Column-Updateability: Y Y Y
Row-Count-Sent:16
1?1SMITH1CLERK1?1? ?w1???S??@1ףp=
?11??? ?w11K1ALLENSALESMAN11?
?w1??K7 ?@1ףp=
?11??? ?w11a1WARDSALESMAN11?
?w1-???'??@1ףp=
?11??? ?w11?1JONES1MANAGER1?1?
?w1?n?>?@1ףp=

# FETCH-RESULT

Syntax:
Command-ID SP FETCH-RESULT CRLF
STATEMENT-ID SP : SP Statement ID CRLF
COMMAND-INDEX SP : SP Command index CRLF
FIRST-ROW-INDEX SP : SP Index of the first row to fetch CRLF
LAST-ROW-INDEX SP : SP Index of the last row to fetch CRLF
OUTPUT-MODE SP : SP RELEASE CRLF CRLF
Binary data

Example:
123 FETCH-RESULT
STATEMENT-ID : 23
COMMAND-INDEX : 3
FIRST-ROW-INDEX : 200
LAST-ROW-INDEX : 299
OUTPUT-MODE : RELEASE

# BULK-MODIFY

Syntax:
Command-ID SP BULK-MODIFY CRLF
STATEMENT-ID SP : SP Statement ID CRLF
COMMAND-INDEX SP : SP Command index CRLF
ROW-COUNT SP : SP Amount of records to be modified CRLF CRLF
Binary data

# Binary mode

- Status byte for each value '1' (non-null), '0' (null), '2' (error). These status codes are characters (not byte values).
- If null then no value, if error then next value is error code VLong8 and no further data is send. If not null then followed by value in binary format.
- Optional row number (32-bit integer)
- VK_STRING are encoded as UTF-16 LE. They start with a 32-bit signed value which is the length (but as a negative number).
- VK_REAL is encoded as a 8-byte double
- VK_TIME (date): 16-bit signed int (year), 1 byte (month), 1 byte (day), 32-bit long (seconds since midnight
)
- VK_BOOLEAN: 16-bit int
- VK_LONG 32-bit int
- VK_BLOB: ???
- VK_IMAGE: ???

Typical response:
1 <id 4 bytes> 1  0xF6FFFFFF UTF16-string 0xF8FFFFFF 



Rows are output from first to last.
If result-set is updateable then each row starts with a 4 byte ID of a record, otherwise record ID is omitted.
Column values within a row are output from left to right.
Each value is preceded by a status byte. If the value was successfully calculated then the status byte is character ‘1’ for non-null values and character ‘0’ for null values.
If there was an error during value calculation then the status byte is character ‘2’.
If the value is null then immediately after ‘0’ status byte goes the data for the next value.
If the status byte is ‘2’ then the status byte is followed by the VLong8 error code and no more data is sent for this command (TODO: Should also send the error message).
The ‘1’ status byte is followed by the binary representation of one of the VValueSingle subclass instances.
The exact type of the instance can be determined using “Column-Types” header.
Binary data for VValueSingle should be read using ReadFromStream() method with the appropriate sub-class of VStream whenever possible.
The only exception from this case is binary values for columns of type VK_IMAGE. VK_IMAGE values are streamed as VBlob and thus should be read as such. Binary data should never be read manually unless XToolBox streaming classes are not available.
Binary data ends with the last value in the last row. There is no command terminator of any kind.

