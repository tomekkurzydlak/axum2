Sub LoadDictionaries()

    Dim basePath As String
    Dim filePath As String
    Dim dictionarySheet As Worksheet
    Dim mainSheet As Worksheet
    Dim validationRange As Range
    Dim lastRow As Long
    Dim queryTable As QueryTable
    Dim filesToLoad As Variant
    Dim i As Long
    Dim targetColumn As Long
    Dim hasAnyFile As Boolean
    Dim validationColumn As Long

    ' Dynamic user path
    basePath = Environ("APPDATA") & "\aaa\dictionaries\"

    Set dictionarySheet = ThisWorkbook.Sheets("DICTIONARIES")
    Set mainSheet = ThisWorkbook.Sheets("aaa")

    Application.ScreenUpdating = False


    filesToLoad = Array( _
        Array("glossary_terms.csv", 31), _
        Array("glossary_terms_2.csv", 32), _
        Array("glossary_terms_3.csv", 33), _
        Array("glossary_terms_4.csv", 34), _
        Array("glossary_terms_5.csv", 35), _
        Array("glossary_terms_6.csv", 36) _
    )

    ' Check whether at least one file exists
    hasAnyFile = False

    For i = LBound(filesToLoad) To UBound(filesToLoad)

        filePath = basePath & CStr(filesToLoad(i)(0))

        If Dir(filePath) <> "" Then
            hasAnyFile = True
            Exit For
        End If

    Next i

    ' Load files only if at least one exists
    If hasAnyFile Then

        ' Remove old QueryTables if they exist from the previous macro version
        For Each queryTable In dictionarySheet.QueryTables
            queryTable.Delete
        Next queryTable

        ' Load each CSV file into its target column
        For i = LBound(filesToLoad) To UBound(filesToLoad)

            filePath = basePath & CStr(filesToLoad(i)(0))
            targetColumn = CLng(filesToLoad(i)(1))

            If Dir(filePath) <> "" Then

                ' Clear only the target column
                dictionarySheet.Columns(targetColumn).Clear

                ' Load UTF-8 BOM CSV without QueryTable.Refresh
                LoadUtf8CsvIntoColumn filePath, dictionarySheet, targetColumn

            Else
                ' Missing file = do nothing.
                ' Existing worksheet data remains unchanged.
            End If

        Next i

    Else
        ' No files found = do nothing.
        ' Existing worksheet data remains unchanged.
    End If

    ' Dictionary range for validation.
    ' Validation still uses column 31 / AE, same as before.
    validationColumn = 31

    lastRow = dictionarySheet.Cells(dictionarySheet.Rows.Count, validationColumn).End(xlUp).Row

    If lastRow >= 2 Then

        Set validationRange = dictionarySheet.Range( _
            dictionarySheet.Cells(2, validationColumn), _
            dictionarySheet.Cells(lastRow, validationColumn))

        ' Set dropdown validation in column M
        With mainSheet.Range("M:M").Validation
            .Delete
            .Add Type:=xlValidateList, _
                 AlertStyle:=xlValidAlertStop, _
                 Operator:=xlBetween, _
                 Formula1:="=" & dictionarySheet.Name & "!" & validationRange.Address
        End With

    End If

    Application.ScreenUpdating = True

End Sub


Private Sub LoadUtf8CsvIntoColumn(ByVal filePath As String, ByVal targetSheet As Worksheet, ByVal targetColumn As Long)

    Dim textStream As Object
    Dim fileText As String
    Dim lines As Variant
    Dim outputData() As Variant
    Dim i As Long
    Dim lineCount As Long
    Dim currentLine As String

    Set textStream = CreateObject("ADODB.Stream")

    With textStream
        .Type = 2
        .Charset = "utf-8"
        .Open
        .LoadFromFile filePath
        fileText = .ReadText
        .Close
    End With

    ' Remove BOM if Excel keeps it in the first value
    fileText = Replace(fileText, ChrW(&HFEFF), "")

    ' Normalize line endings
    fileText = Replace(fileText, vbCrLf, vbLf)
    fileText = Replace(fileText, vbCr, vbLf)

    lines = Split(fileText, vbLf)

    ReDim outputData(1 To UBound(lines) + 1, 1 To 1)

    lineCount = 0

    For i = LBound(lines) To UBound(lines)

        currentLine = CStr(lines(i))

        If Len(Trim(currentLine)) > 0 Then
            lineCount = lineCount + 1
            outputData(lineCount, 1) = CleanCsvValue(currentLine)
        End If

    Next i

    If lineCount > 0 Then
        targetSheet.Cells(1, targetColumn).Resize(lineCount, 1).Value = outputData
    End If

End Sub


Private Function CleanCsvValue(ByVal inputText As String) As String

    inputText = Trim(inputText)

    ' If the CSV has one column but values are wrapped in quotes
    If Left(inputText, 1) = """" And Right(inputText, 1) = """" Then
        inputText = Mid(inputText, 2, Len(inputText) - 2)
    End If

    ' Convert escaped CSV double quotes into normal quotes
    inputText = Replace(inputText, """""", """")

    CleanCsvValue = inputText

End Function
