# PPU Verilog

В данном разделе приводятся выдержки из исходников реализации PPU на верилоге.

Если вы прочитали всю книгу и сразу бросились делать PPU на верилоге - остановитесь. Уже все сделано :smile:

Исходники публикуются как есть, все замечания можно отправлять через репу гитхаба (где они скоро будут также опубликованы в полном составе), либо через дискорд.

```verilog
// Управление модулями по адресу
module PPU_ADR_CONTROL(
	// Системные входы
	input	Clk,			// Системные такты
	input	PClk,			// Позитивные такты
	input	ALEGate,		// Формирователь сигнала ALE
	// Управляющие входы
	input	W7,				// Запись в регистр 7
	input	R7,				// Чтение из регистра 7
	input	H0n,			// Слот обращения
	input	BLNK,			// Картинка погашена
	input	PA8,			// Адрес PA8
	input	PA9,			// Адрес PA9
	input	PA10,			// Адрес PA10
	input	PA11,			// Адрес PA11
	input	PA12,			// Адрес PA12
	input	PA13,			// Адрес PA13
	// Выходы
	output	PWR,			// Активация записи
	output	TSTEP,			// Счет развертки
	output	DB_PAR,			// Данные на шину PPU
	output	PD_RB,			// Данные в защелку
	output	ALE,			// Сигнал ALE
	output	PRD,			// Активация чтения
	output	XRB,			// Данные на шину CPU
	output	TH_MUX			// Обращение в палитру
);
// Переменные
assign TH_MUX = BLNKR & PA8 & PA9 & PA10 & PA11 & PA12 & PA13;
reg [1:0]W7Req;				// Очередь запроса записи
reg [1:0]W7Queue;			// Очередь сигнала записи
reg [1:0]R7Req;				// Очередь запроса чтения
reg [1:0]R7Queue;			// Очередь сигнала чтения
reg BLNKR;					// Регистр гашения
// Выход записи
assign DB_PAR = W7Queue[1] & ~W7Queue[0];
assign PWR = TH_MUX | ~DB_PAR;
// Выход чтения
assign PD_RB = R7Queue[1] & ~R7Queue[0];
assign PRD = ~(PD_RB | (H0n & ~BLNKR));
// Выход шага счетчика растра
assign TSTEP = DB_PAR | PD_RB;
// Выход ALE
assign ALE = (~W7Queue[1] & W7Queue[0]) | (~R7Queue[1] & R7Queue[0]) | (ALEGate & ~H0n & ~BLNK);
// Данные на шину CPU
assign XRB = R7 & ~TH_MUX;
// Логика
always @(posedge Clk) begin
	// Синхронизация
	W7Req[0] <= W7; R7Req[0] <= R7;
	// Запросы
	if (W7Req[0] & ~W7) W7Req[1] <= 1'b1;
		else if (W7Queue[1]) W7Req[1] <= 1'b0;
	if (R7Req[0] & ~R7) R7Req[1] <= 1'b1;
		else if (R7Queue[1]) R7Req[1] <= 1'b0;
	// Пиксельклок
	if (PClk) begin
		// Ротация очереди
		W7Queue[1:0] <= {W7Queue[0],W7Req[1]};
		R7Queue[1:0] <= {R7Queue[0],R7Req[1]};
		// Синхронизация
		BLNKR <= BLNK;
		end
end
// конец модуля
endmodule
```

```verilog
//===============================================================================================
// FIFO выборки спрайтов
//===============================================================================================
module OAM_FIFO(
	// Такты
	input	Clk,				// Тактовая частота
	input	PClk,				// Пиксельклок
	// Управление FIFO
	input	OVZn,				// Обнуление графики спрайтов
	input	O_HPOS,				// Нулевая координата по горизонтали
	// Управление растром
	input	CLIP_O,				// Кроп левого столбца
	input	VIS,				// Гашение
	input	OBE,				// Активация спрайтов
	// Атомарное состояние
	input	PAR_O,				// Выборка данных спрайтов
	input	[5:0]Hnn,			// Номер состояния
	// Исходные данные
	input	[7:0]OB,			// Данные атрибутов спрайтов
	input	[7:0]PAi,			// Данные графики спрайтов
	// Выходные данные
	output	[4:0]ZCOL,			// Графические данные спрайтов
	output	reg SPR0HIT,		// Детектор спрайта #0
	output	SH2					// Чтение атрибутов спрайтов (для мирроринга по вертикали)
);
// Комбинаторика
assign SH2 = PAR_O & ~Hnn[2] & Hnn[1] & ~Hnn[0];
wire [7:0]GD;					// Графические данные для сдвига
assign GD[0] = (Mirr) ? PAi[7] & OVZn : PAi[0] & OVZn;
assign GD[1] = (Mirr) ? PAi[6] & OVZn : PAi[1] & OVZn;
assign GD[2] = (Mirr) ? PAi[5] & OVZn : PAi[2] & OVZn;
assign GD[3] = (Mirr) ? PAi[4] & OVZn : PAi[3] & OVZn;
assign GD[4] = (Mirr) ? PAi[3] & OVZn : PAi[4] & OVZn;
assign GD[5] = (Mirr) ? PAi[2] & OVZn : PAi[5] & OVZn;
assign GD[6] = (Mirr) ? PAi[1] & OVZn : PAi[6] & OVZn;
assign GD[7] = (Mirr) ? PAi[0] & OVZn : PAi[7] & OVZn;
wire [7:0]SPRIO;				// Приоритет спрайтов
assign SPRIO[0] = (Done[0] & VIS) & VIS & OBE & ~CLIP_O & (COL1[0] | COL0[0]);
assign SPRIO[1] = (Done[1] & VIS) & VIS & OBE & ~CLIP_O & (COL1[1] | COL0[1]);
assign SPRIO[2] = (Done[2] & VIS) & VIS & OBE & ~CLIP_O & (COL1[2] | COL0[2]);
assign SPRIO[3] = (Done[3] & VIS) & VIS & OBE & ~CLIP_O & (COL1[3] | COL0[3]);
assign SPRIO[4] = (Done[4] & VIS) & VIS & OBE & ~CLIP_O & (COL1[4] | COL0[4]);
assign SPRIO[5] = (Done[5] & VIS) & VIS & OBE & ~CLIP_O & (COL1[5] | COL0[5]);
assign SPRIO[6] = (Done[6] & VIS) & VIS & OBE & ~CLIP_O & (COL1[6] | COL0[6]);
assign SPRIO[7] = (Done[7] & VIS) & VIS & OBE & ~CLIP_O & (COL1[7] | COL0[7]);
assign ZCOL[4:0] =	(SPRIO[0]) ? {PRIO[0],CA1[0],CA0[0],COL1[0],COL0[0]} : 
					(SPRIO[1]) ? {PRIO[1],CA1[1],CA0[1],COL1[1],COL0[1]} : 
					(SPRIO[2]) ? {PRIO[2],CA1[2],CA0[2],COL1[2],COL0[2]} : 
					(SPRIO[3]) ? {PRIO[3],CA1[3],CA0[3],COL1[3],COL0[3]} : 
					(SPRIO[4]) ? {PRIO[4],CA1[4],CA0[4],COL1[4],COL0[4]} : 
					(SPRIO[5]) ? {PRIO[5],CA1[5],CA0[5],COL1[5],COL0[5]} : 
					(SPRIO[6]) ? {PRIO[6],CA1[6],CA0[6],COL1[6],COL0[6]} : 
					(SPRIO[7]) ? {PRIO[7],CA1[7],CA0[7],COL1[7],COL0[7]} : 
					5'h00;
// Переменные
reg [7:0]CA0;					// Управление загрузкой атрибута 0
reg [7:0]CA1;					// Управление загрузкой атрибута 1
reg [7:0]PRIO;					// Управление приоритетом
wire [7:0]Done;					// Разрешение вывода графики
reg Mirr;						// Горизонтальное отражение спрайтов
wire [7:0]COL0;					// Выход графики первого байта спрайта
wire [7:0]COL1;					// Выход графики второго байта спрайта
// Счетчики горизонтальной позиции
HPOS_COUNTER HPosCounter0(Clk, PClk, ~Hnn[5] & ~Hnn[4] & ~Hnn[3] & ~Hnn[2] & Hnn[1] & Hnn[0] & PAR_O, O_HPOS, OB[7:0], Done[0]); 
HPOS_COUNTER HPosCounter1(Clk, PClk, ~Hnn[5] & ~Hnn[4] &  Hnn[3] & ~Hnn[2] & Hnn[1] & Hnn[0] & PAR_O, O_HPOS, OB[7:0], Done[1]); 
HPOS_COUNTER HPosCounter2(Clk, PClk, ~Hnn[5] &  Hnn[4] & ~Hnn[3] & ~Hnn[2] & Hnn[1] & Hnn[0] & PAR_O, O_HPOS, OB[7:0], Done[2]); 
HPOS_COUNTER HPosCounter3(Clk, PClk, ~Hnn[5] &  Hnn[4] &  Hnn[3] & ~Hnn[2] & Hnn[1] & Hnn[0] & PAR_O, O_HPOS, OB[7:0], Done[3]); 
HPOS_COUNTER HPosCounter4(Clk, PClk,  Hnn[5] & ~Hnn[4] & ~Hnn[3] & ~Hnn[2] & Hnn[1] & Hnn[0] & PAR_O, O_HPOS, OB[7:0], Done[4]); 
HPOS_COUNTER HPosCounter5(Clk, PClk,  Hnn[5] & ~Hnn[4] &  Hnn[3] & ~Hnn[2] & Hnn[1] & Hnn[0] & PAR_O, O_HPOS, OB[7:0], Done[5]); 
HPOS_COUNTER HPosCounter6(Clk, PClk,  Hnn[5] &  Hnn[4] & ~Hnn[3] & ~Hnn[2] & Hnn[1] & Hnn[0] & PAR_O, O_HPOS, OB[7:0], Done[6]); 
HPOS_COUNTER HPosCounter7(Clk, PClk,  Hnn[5] &  Hnn[4] &  Hnn[3] & ~Hnn[2] & Hnn[1] & Hnn[0] & PAR_O, O_HPOS, OB[7:0], Done[7]);
// Регистры сдвига графики
SHIFT_REG ShiftSp0A(Clk, PClk, ~Hnn[5] & ~Hnn[4] & ~Hnn[3] & Hnn[2] & ~Hnn[1] & Hnn[0] & PAR_O, Done[0] & VIS, GD[7:0], COL0[0]);
SHIFT_REG ShiftSp0B(Clk, PClk, ~Hnn[5] & ~Hnn[4] & ~Hnn[3] & Hnn[2] &  Hnn[1] & Hnn[0] & PAR_O, Done[0] & VIS, GD[7:0], COL1[0]);
SHIFT_REG ShiftSp1A(Clk, PClk, ~Hnn[5] & ~Hnn[4] &  Hnn[3] & Hnn[2] & ~Hnn[1] & Hnn[0] & PAR_O, Done[1] & VIS, GD[7:0], COL0[1]);
SHIFT_REG ShiftSp1B(Clk, PClk, ~Hnn[5] & ~Hnn[4] &  Hnn[3] & Hnn[2] &  Hnn[1] & Hnn[0] & PAR_O, Done[1] & VIS, GD[7:0], COL1[1]);
SHIFT_REG ShiftSp2A(Clk, PClk, ~Hnn[5] &  Hnn[4] & ~Hnn[3] & Hnn[2] & ~Hnn[1] & Hnn[0] & PAR_O, Done[2] & VIS, GD[7:0], COL0[2]);
SHIFT_REG ShiftSp2B(Clk, PClk, ~Hnn[5] &  Hnn[4] & ~Hnn[3] & Hnn[2] &  Hnn[1] & Hnn[0] & PAR_O, Done[2] & VIS, GD[7:0], COL1[2]);
SHIFT_REG ShiftSp3A(Clk, PClk, ~Hnn[5] &  Hnn[4] &  Hnn[3] & Hnn[2] & ~Hnn[1] & Hnn[0] & PAR_O, Done[3] & VIS, GD[7:0], COL0[3]);
SHIFT_REG ShiftSp3B(Clk, PClk, ~Hnn[5] &  Hnn[4] &  Hnn[3] & Hnn[2] &  Hnn[1] & Hnn[0] & PAR_O, Done[3] & VIS, GD[7:0], COL1[3]);
SHIFT_REG ShiftSp4A(Clk, PClk,  Hnn[5] & ~Hnn[4] & ~Hnn[3] & Hnn[2] & ~Hnn[1] & Hnn[0] & PAR_O, Done[4] & VIS, GD[7:0], COL0[4]);
SHIFT_REG ShiftSp4B(Clk, PClk,  Hnn[5] & ~Hnn[4] & ~Hnn[3] & Hnn[2] &  Hnn[1] & Hnn[0] & PAR_O, Done[4] & VIS, GD[7:0], COL1[4]);
SHIFT_REG ShiftSp5A(Clk, PClk,  Hnn[5] & ~Hnn[4] &  Hnn[3] & Hnn[2] & ~Hnn[1] & Hnn[0] & PAR_O, Done[5] & VIS, GD[7:0], COL0[5]);
SHIFT_REG ShiftSp5B(Clk, PClk,  Hnn[5] & ~Hnn[4] &  Hnn[3] & Hnn[2] &  Hnn[1] & Hnn[0] & PAR_O, Done[5] & VIS, GD[7:0], COL1[5]);
SHIFT_REG ShiftSp6A(Clk, PClk,  Hnn[5] &  Hnn[4] & ~Hnn[3] & Hnn[2] & ~Hnn[1] & Hnn[0] & PAR_O, Done[6] & VIS, GD[7:0], COL0[6]);
SHIFT_REG ShiftSp6B(Clk, PClk,  Hnn[5] &  Hnn[4] & ~Hnn[3] & Hnn[2] &  Hnn[1] & Hnn[0] & PAR_O, Done[6] & VIS, GD[7:0], COL1[6]);
SHIFT_REG ShiftSp7A(Clk, PClk,  Hnn[5] &  Hnn[4] &  Hnn[3] & Hnn[2] & ~Hnn[1] & Hnn[0] & PAR_O, Done[7] & VIS, GD[7:0], COL0[7]);
SHIFT_REG ShiftSp7B(Clk, PClk,  Hnn[5] &  Hnn[4] &  Hnn[3] & Hnn[2] &  Hnn[1] & Hnn[0] & PAR_O, Done[7] & VIS, GD[7:0], COL1[7]);
// Логика
always @(posedge Clk) begin
	if (PClk) begin
		// Загрузка атрибутов спрайта
		if (PAR_O) begin
			if (~Hnn[5] & ~Hnn[4] & ~Hnn[3] & ~Hnn[2] & Hnn[1] & ~Hnn[0] & PAR_O) {PRIO[0],CA1[0],CA0[0]} <= {OB[5],OB[1:0]};
			if (~Hnn[5] & ~Hnn[4] &  Hnn[3] & ~Hnn[2] & Hnn[1] & ~Hnn[0] & PAR_O) {PRIO[1],CA1[1],CA0[1]} <= {OB[5],OB[1:0]};
			if (~Hnn[5] &  Hnn[4] & ~Hnn[3] & ~Hnn[2] & Hnn[1] & ~Hnn[0] & PAR_O) {PRIO[2],CA1[2],CA0[2]} <= {OB[5],OB[1:0]};
			if (~Hnn[5] &  Hnn[4] &  Hnn[3] & ~Hnn[2] & Hnn[1] & ~Hnn[0] & PAR_O) {PRIO[3],CA1[3],CA0[3]} <= {OB[5],OB[1:0]};
			if ( Hnn[5] & ~Hnn[4] & ~Hnn[3] & ~Hnn[2] & Hnn[1] & ~Hnn[0] & PAR_O) {PRIO[4],CA1[4],CA0[4]} <= {OB[5],OB[1:0]};
			if ( Hnn[5] & ~Hnn[4] &  Hnn[3] & ~Hnn[2] & Hnn[1] & ~Hnn[0] & PAR_O) {PRIO[5],CA1[5],CA0[5]} <= {OB[5],OB[1:0]};
			if ( Hnn[5] &  Hnn[4] & ~Hnn[3] & ~Hnn[2] & Hnn[1] & ~Hnn[0] & PAR_O) {PRIO[6],CA1[6],CA0[6]} <= {OB[5],OB[1:0]};
			if ( Hnn[5] &  Hnn[4] &  Hnn[3] & ~Hnn[2] & Hnn[1] & ~Hnn[0] & PAR_O) {PRIO[7],CA1[7],CA0[7]} <= {OB[5],OB[1:0]};
			end
		// Горизонтальное отражение
		if (PAR_O & ~Hnn[2] & Hnn[1] & ~Hnn[0]) Mirr <= OB[6];
		// Детектор вывода спрайта #0
		SPR0HIT <= SPRIO[0];
		end
end
// Конец модуля OAM_FIFO
endmodule
```

```verilog
//===============================================================================================
// Генератор фона
//===============================================================================================
module BG_GEN(
	// Такты
	input	Clk,			// Тактовый вход
	input	PClk,			// Пиксельклок
	// Атомарное состояние
	input	H0nn,			// Активность шины
	input	F_AT,			// Фаза выборки атрибутов
	input	F_TA,			// Фаза выборки первого байта тайла
	input	F_TB,			// Фаза выборки второго байта атрибутов
	input	N_FO,			// Активация сдвига графики
	// Управление растром
	input	BGE,			// Включение фона
	input	VIS,			// Гашение
	input	CLIP_B,			// Кроп левого столбца
	// Управление атрибутами
	input	THO1,			// Горизонтальная координата в атрибуте
	input	TVO1,			// Вертиклаьная координата в атрибуте
	// Управление сдвигом
	input	W5_1,			// Запись в регистр горизонтальной прокрутки
	input	[2:0]DIn,		// Данные прокрутки
	// Шина данных графики
	input	[7:0]PAi,		// Графические данные
	// Выход картинки
	output	reg [3:0]BGC	// Выход пикселей
);
// Переменные
reg THO1R;					// Синхронизация горизонтальной координаты атрибутов
reg TVO1R;					// Синхронизация вертикальной координаты атрибутов
reg [7:0]ATR;				// Регистр хранения атрибута
reg [7:0]TAR;				// Регистр хранения первого байта тайла
reg [1:0]ATS;				// Хранение атрибута для сдвига
reg [7:0]TAS;				// Сдвиговый регистр первого байта тайла
reg [7:0]TBS;				// Сдвиговый регистр второго байта тайла
reg [7:0]PA0;				// Регистр позиции младшего бита атрибутов
reg [7:0]PA1;				// Регистр позиции старшего бита атрибута
reg [7:0]PTA;				// Регистр позиции первого байта тайла
reg [7:0]PTB;				// Регистр позиции второго байта тайла
reg [2:0]BHP;				// Горизонтальная позиция растра фона
// Комбинаторика, выбор атрибута
wire [1:0]ATSel;
assign ATSel[1:0] = (~TVO1R) ?
						(~THO1R) ? ATR[1:0] : ATR[3:2]
					:
						(~THO1R) ? ATR[5:4] : ATR[7:6];
// Комбинаторика, выход пикселей
wire [3:0]BGCS;
assign BGCS[3:0] = (BGE & VIS & ~CLIP_B) ? 
						(~BHP[2]) ?
							(~BHP[1]) ?
								(~BHP[0]) ? {PA1[0],PA0[0],PTB[0],PTA[0]} : {PA1[1],PA0[1],PTB[1],PTA[1]}
							:
								(~BHP[0]) ? {PA1[2],PA0[2],PTB[2],PTA[2]} : {PA1[3],PA0[3],PTB[3],PTA[3]}
						:
							(~BHP[1]) ?
								(~BHP[0]) ? {PA1[4],PA0[4],PTB[4],PTA[4]} : {PA1[5],PA0[5],PTB[5],PTA[5]}
							:
								(~BHP[0]) ? {PA1[6],PA0[6],PTB[6],PTA[6]} : {PA1[7],PA0[7],PTB[7],PTA[7]}
					: 4'h0;
// Логика
always @(posedge Clk) begin
	if (PClk) begin
		// Синхронизация
		THO1R <= THO1; TVO1R <= TVO1;
		// Чтение с графической шины
		if (H0nn & F_AT) ATR[7:0] <= PAi[7:0];
		if (H0nn & F_TA) TAR[7:0] <= PAi[7:0];
		// Сдвиговый регистр графики
		if (N_FO) begin
			// Если работа разрешена
			if (H0nn & F_TB) begin
				// Загрузка графических данных и атрибутов
				ATS[1:0] <= ATSel[1:0];
				TAS[7:0] <= TAR[7:0];
				TBS[7:0] <= PAi[7:0];
				end else begin
				// Сдвиг графических данных
				TAS[7:0] <= {TAS[6:0],1'b0};
				TBS[7:0] <= {TBS[6:0],1'b0};
				end
			end
		// Регистр позиционирования растра
		if (N_FO) begin
			// Если разрешена работа сдвига
			PA0[7:0] <= {ATS[0],PA0[7:1]};
			PA1[7:0] <= {ATS[1],PA1[7:1]};
			PTA[7:0] <= {TAS[7],PTA[7:1]};
			PTB[7:0] <= {TBS[7],PTB[7:1]};
			end
		// Регистр горизонтальной позиции растра
		if (W5_1) BHP[2:0] <= DIn[2:0];
		// Синхронизация видеовыхода
		BGC[3:0] <= BGCS[3:0];
		end
end
// Конец модуля BG_GEN
endmodule
```
