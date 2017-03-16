# Bootloader_F4x
Загрузчик стартует с адреса FLASHBOOT_SECTOR (0x08000000 во flash.). В секторе FLAG_STATUS_SECTOR(0x08004000) перебирает байты пока не дойдет до
неписанного поля 0xff. Затем читает значение предыдущего байта.

Если флаг 0xA7
* Приложение приняло прошивку во вторую половину флэш в рабочем режиме и 
 требуется переписать ее в первую половину в режиме загрузчика.
* по адресу FIRM_UPD_SECTOR(0x08080000)+0x1C считаем 4 байта размера прошивки bin_size
* проверить crc принятой обновленной прошивки во второй половине флэш сравнить со значениемпо адресу  FIRM_UPD_SECTOR+bin_size
*	переписать со второй половины флэш в первую(рабочую) 
* записать в сектор FLAG_STATUS_SECTOR  флаг 0xA3 следующим за считанным  0xA7(в чистом месте 0xFF) 
* сделать reset для запуска обновленной прошивки из первой половины флэш.

Если флаг 0xA3.
* Обновление не требуется и загрузчик после проверки целостности прошивки в рабочей части флэш передает управление приложению
* по адресу FIRM_WORK_SECTOR(0x08008000)+0x1C считаем 4 байта размера прошивки bin_size
* проверить crc прошивки и сравнить со значение которое лежит по адресу FIRM_WORK_SECTOR+bin_size
* если crc верное 
    * Передвинуть таблицу векторов на FLASH_BASE+FLASHBOOT_SIZE 		(0x08000000+0x8000)
    * Установить указатель стэка SP значением с адреса FIRM_WORK_SECTOR
    * Запустить функцию (приложение ) по адресу взятому с (FIRM_WORK_SECTOR+4)
 * если crc не верное запустить функцию Bootloader_upd_firmware() которая ожидает прошивку и принимает  ее по CAN сети
 
 Если флаг 0xFF
 * или другое любое значение кроме 0xA3 0xA7  также запускается Bootloader_upd_firmware()
 
 Функция Bootloader_upd_firmware()
* Настраивает PF7 для моргания светодиодом во время принятия прошивки по CAN
* Отправляет запрос по сети на выдачу прошивки CAN_Data_TX.ID=(NETNAME_INDEX<<8)|0x74;(GET_FIRMWARE)  CAN_Data_TX.Data[0]=NETNAME_INDEX;
*  Ждет сброса флага get_firmware_size=0
* Подготавливает флэш для записи (разблок. и стирает если нужно)
* Отправляет первый запрос CAN_Data_TX.ID=(NETNAME_INDEX<<8)|0x72;(GET_DATA)
* Далее запросы отправляются из ISR после принятия очередной порции байт (8 байт)   
* Ждем окончания принятия прошивки ( установка write_flashflag=1)
* Записывает в сектор FLAG_STATUS_SECTOR флаг 0xA3 по адресу  (FLAG_STATUS_SECTOR+count) затем небольшая пауза и RESET контроллера.

Модуль CAN настраивается:
* Автоматич. ретрансляция включена
* Контроллер выходит из состояния «Bus-Off» автоматически 
* tBS1=tq*(7+1)=8*tq
* tBS2=tq*(2+1)=3*tq
* Sample point = 75%
* f=125kHz
* Filters bank 3 4 5  mode ID List
* Filters bank 3 4 5  scale 16 bits	
* Filters bank 3 4 5 FIFO1	
* ID=0x271 IDE=0 RTR=0	//  SET_FIRMWARE_SIZE
* ID=0x273 IDE=0 RTR=0	// SET_DATA_FIRMWARE
* ID=0x088 IDE=0 RTR=1	// GET_NET_NAME	 
 

