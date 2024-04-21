# bash_script_folder_hash_calculator
Калькулятор hash суммы для windows power shell

Чтобы запустить продсчёт контрольной суммы в папках (суммарно по всем входящим в папку объектам, включая вложенные папки) нужно перейти в корневую папку относительно тех, для которых производится расчёт и запустить скрипт ниже. 
В вызове функции подать в неё названия папок в кавычках через запятую.
В коде ниже это "VM_212", "VM_216", "VM_658", "VM_800"


********************************************************
#Вариант 1 кода на одном ядре процессора. Работает
`
#Подсчёт для всех объектов последовательно один за другим
function Get-FolderHash {
    param (
        [Parameter(Mandatory=$true)]
        [string[]]$FolderNames
    )

    # Проходим по каждой папке из списка
    foreach ($folderName in $FolderNames) {
        # Собираем полный путь к папке
        $fullPath = Join-Path -Path (Get-Location) -ChildPath $folderName
        $fullPath
	
        # Получаем список файлов в папке
        $fileList = Get-ChildItem -Path $fullPath -Recurse -File
        $totalFiles = $fileList.Count

        # Инициализируем переменную для хранения общей хеш-суммы
        $combinedHashString = ""

        # Инициализируем переменные для отслеживания прогресса
        $processedFiles = 0

        # Начинаем обработку файлов
        foreach ($file in $fileList) {
            # Обновляем счетчик обработанных файлов
            $processedFiles++
            
            # Вычисляем процент завершенности
            $percentComplete = ($processedFiles / $totalFiles) * 100
            
            # Отображаем прогресс
            Write-Progress -Activity "Processing files in $folderName" -Status "Processed $processedFiles of $totalFiles files. $percentComplete'% done " -PercentComplete $percentComplete
            
            # Получаем хеш-сумму файла
            $fileHash = Get-FileHash -Path $file.FullName -Algorithm MD5
            
            # Добавляем хеш-сумму файла к общей хеш-сумме
            $combinedHashString += $fileHash.Hash
        }

        # Вычисляем общую хеш-сумму из всех хешей
        $combinedHash = Get-FileHash -Algorithm MD5 -InputStream ([System.IO.MemoryStream]::new([System.Text.Encoding]::UTF8.GetBytes($combinedHashString)))

        # Выводим общую хеш-сумму для текущей папки
        Write-Host "Контрольная сумма для $folderName = $($combinedHash.Hash)"
    }
}

#Начало исполняемого кода:
#Очищаем консоль
clear

#Вызываем функцию с указанием папок для вычисления хеш-суммы
Get-FolderHash -FolderNames "VM_212", "VM_216", "VM_658", "VM_800" #Вызываем основную функцию Get-FolderHash
#Конец исполняемого кода
`



*********************************************************************
#Вариант 2 кода на разных ядрах процессора. Требует отладки

`
function Get-FolderHash {
    param (
        [Parameter(Mandatory=$true)]
        [string[]]$FolderNames
    )
    
    $currentFolder = Get-Location

    #Создаем обработчик события для обновления прогресс-баров
    $global:progressEventHandler = {
        param($Sender, $Event)
        Write-Progress -Activity "Processing files in $($Event.SourceIdentifier)" -Status "$($Event.Message)" -PercentComplete $Event.Progress
    }

    #Проходим по каждой папке из списка
    foreach ($folderName in $FolderNames) {
        #Создаем событие для текущей папки
        $event = Register-ObjectEvent -SourceIdentifier "FolderProgress_$folderName" -EventName "ProgressUpdated" -Action $global:progressEventHandler
        
        Start-Job -ScriptBlock {
            param ($folderName, $currentFolder, $event)
            
            # Cобираем полный путь к папке
            $fullPath = Join-Path -Path $currentFolder -ChildPath $folderName
            Write-Output "Full path for $folderName\: $fullPath"

            #Получаем список файлов в папке
            $fileList = Get-ChildItem -Path $fullPath -Recurse -File
            $totalFiles = $fileList.Count

            #Инициализируем переменную для хранения общей хеш-суммы
            $combinedHashString = ""

            #Инициализируем переменные для отслеживания прогресса
            $processedFiles = 0

            #Начинаем обработку файлов
            foreach ($file in $fileList) {
                #Обновляем счетчик обработанных файлов
                $processedFiles++
                
                #Вычисляем процент завершенности
                $percentComplete = ($processedFiles / $totalFiles) * 100
                
                #Вызываем событие для обновления прогресса
                $event.Message = "Processed $processedFiles of $totalFiles files"
                $event.Progress = $percentComplete
                $event.Raise()
                
                #Получаем хеш-сумму файла
                $fileHash = Get-FileHash -Path $file.FullName -Algorithm MD5
                
                #Добавляем хеш-сумму файла к общей хеш-сумме
                $combinedHashString += $fileHash.Hash
            }

            #Возвращаем общую хеш-сумму для текущей папки
            $combinedHash = Get-FileHash -Algorithm MD5 -InputStream ([System.IO.MemoryStream]::new([System.Text.Encoding]::UTF8.GetBytes($combinedHashString)))
            return "Контрольная сумма для $folderName = $($combinedHash.Hash)"
        } -ArgumentList $folderName, $currentFolder, $event | Out-Null
    }

    #Ожидаем завершения всех заданий и выводим результаты
    $result = @(Get-Job | Wait-Job | Receive-Job)
    Remove-Job -State Completed

    #Удаляем обработчик события
    Unregister-Event -SourceIdentifier "FolderProgress_*"

    # Выводим результаты
    $result
}
#Начало исполняемого кода:
#Очищаем консоль
clear

#Вызываем функцию с указанием папок для вычисления хеш-суммы
Get-FolderHash -FolderNames "VM_212", "VM_216", "VM_658", "VM_800" #Вызываем основную функцию Get-FolderHash
#Конец исполняемого кода
`
