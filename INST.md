> [!CAUTION]
> PROJETO INCOMPLETO!

Acredito que esteja 95%+ finalizado. Falta apenas documentar a lógica de leitura do fluxostato no canal A1 do adc e modificar seu respectivo switch(flowswitchbn). 

*Desconsidere o esquema do fluxostato.

No momento não pretendo levar a ideia adiante. Fiz testes reais e os resultados foram satisfatórios. Apesar desse projeto ter sido feito por um amador, eu acredito que seja uma base sólida.

# Estrutura

A seguinte estrutura de arquivos é necessária para compilar:

```
presscontrol/
   ├─ fonts/
   │  ├─ materialdesignicons-webfont.ttf
   │  └─ Nunito-SemiBold.ttf
   ├─ pressureswitchs3.yml
   ├─ lvgl.yml
   └─ v3.js
   
Onde baixar:
Material Design icon:
https://github.com/Templarian/MaterialDesign-Webfont/blob/master/fonts/materialdesignicons-webfont.ttf
Clique no botão "Download raw file" a direita do botão "Raw".

Nunito:
https://fonts.google.com/specimen/Nunito
Clique no botão azul "Get font" e depois em "Download all";
Após o download, abra e selecione para extrair "Nunito-SemiBold.ttf".

v3.js:
Crie um arquivo chamado v3,js, copie tudo contido em https://oi.esphome.io/v3/www.js cole e salve.
```


# Presscontrol

Esse projeto permite automatizar um motor/bomba baseado na pressão definida pelo usuário.

Inicialmente foi pensado para trabalhar em tubulações com água mas dado o principio de funcionamento do transdutor, é possível usá-lo em tubulações com ar(não testado).

Sensor definido para funcionar em loop de 250ms. Isso significa que a resposta de liga/desliga da bomba pode ter um "atraso" de até 250ms. Esse "atraso" é irrelevante para minha aplicação. Caso você precise de algo que tenha uma resposta imediata, esse projeto não irá atender.

### Hardware

- esp32s3 n16r8*;
- Transdutor de pressão 0-10bar 4-20ma;
- Bucha de redução de 1/4(ou de acordo com seu transdutor) para o diâmetro desejado da tubulação;
- ADS1115 16bit adc;
- Relé SSR/Mecânico dimensionado de acordo com seu motor/bomba;
- Max6675 + Termopar tipo K(opcional);
- Fluxostato de acordo com o tubulação usada(opcional);
- Display LCD TFT de 2.0 Polegadas com Encoder Rotativo EC11;

*É possível usar outros modelos de esp32 mas será preciso modificações no projeto.

### Wiring

```
                        Display LCD TFT de 2.0 Polegadas com Encoder Rotativo EC11
                           ╭――――――――――――――――――――――――――――――――――――――――――――――――╮
                           │    ╭――――――――――――――――――――――――――――――╮     GND [■]│ ---> ■ GND
                           │    │                              │     VDD [■]│ ---> ■ 3.3V
                           │    │                              │     SCL [■]│ ---> ■ GPIO12
                           │    │                              │     SDA [■]│ ---> ■ GPIO11
                           │    │                              │     RES [■]│ ---> ■ GPIO13
                           │    │                              │      DC [■]│ ---> ■ GPIO10
                           │    │                              │      CS [■]│ ---> ■ GND
                           │    │                              │     BLK [■]│ ---> ■ GPIO14
                           │    │                              │     TRA [■]│ ---> ■ GPIO4
                           │    │                              │     TRB [■]│ ---> ■ GPIO5
                           │    │                              │     PSH [■]│ ---> ■ GPIO6
                           │    ╰――――――――――――――――――――――――――――――╯      K0 [■]│ ---> ■ GPIO7
                           │        ╭――╮             ╭――――――╮               │
                           │        |◯|             |  ◯  |               │
                           │        ╰――╯             ╰――――――╯               │
                           │        KEY0               EC11                 │
                           ╰――――――――――――――――――――――――――――――――――――――――――――――――╯

                                ADS1115 16bit ADC
                              ╭――――――――――――――――――――╮
                  3.3V ■ <--- │ [■] VDD ╭―╮      ◯|
                   GND ■ <--- │ [■] GND | | ╭―――╮  |
                 GPIO9 ■ <--- │ [■] SCL ╰―╯ ╰―――╯  |
                 GPIO8 ■ <--- │ [■] SDA     ╭―――╮  |
                              │ [ ] ADDR    ╰―――╯  |
                              │ [ ] ALRT    ╭―――╮  |
      Preto Transdutor ■ <--- │ [■] A0      ╰―――╯  |
  Fluxostato(opcional) ■ <--- │ [■] A1  - ╭―――╮-   |
                              │ [ ] A2  - |   |-   |
                              │ [ ] A3  - ╰―――╯- ◯|
                              ╰――――――――――――――――――――╯

         TRANSDUTOR                                            FONTE                                                              

      ╭――――――――――――――――╮―╮                                  ╭―――――――――――――――╮
      |           |    |\| -------VERMELHO(24V)------------>| +             |
      |           |    |\| ---SINAL(4-20ma)-╮               |    24V DC     |
      |         ╭――――――╯―╯                  |           ╭-->| -             |
      |╭―――――――╮|                           |           |   ╰―――――――――――――――╯   
      ||0~10bar||                           |           |                               
      |╰―――――――╯|                           |           |----- ESP32 GND                             
      |         |                           |--/\/\/\/\-╯                                        
      |―――――――――|                           |    150                                    
      | \/   \/ |                           |     
      | /\   /\ |                           |
      ╰―――――――――╯                           |
        |=====|                         A0 ADS1115                           
        |=====|                                                                 
        ╰―――――╯                                                           
                                                                
                              


                     RELÉS SSR OU MECÂNICOS PODEM SER USADOS
                                                                      
           RElÉ DE ESTADO SOLIDO                        RELÉ MECANICO                  
           ╭―――――――╮   ╭――――――――╮              ╭――――――――――――――――――――――――――――――――╮                                                   
           |  \/   ╰―――╯   \/   |              |◯ [■■][]    ╭――――――――――╮     ◯|           
           |  /\           /\   |      5V   <--| ■ VCC | | | |          |  NC ■ |           
           |――――――――――――――――――――|    GPIO38 <--| ■ IN  ╭――――╮|          | COM ■ |          
           |  ╰――――|   |――――╯   | ESP32 GND <--| ■ GND ╰――――╯|          |  NO ■ |           
           |   SSR-XX           |              |◯     | | | ╰――――――――――╯     ◯|
           |   ╭―INPUT―――◯―╮   |              ╰――――――――――――――――――――――――――――――――╯                                    
           |   -            +   |                             
           |――――――――――――――――――――|                                          
           |  \/           \/   |                                          
           |  /\   ╭―――╮   /\   |                                          
           ╰―――――――╯   ╰――――――――╯                                                                         
               |            |                                              
               |            |                                                          
               |            |                                                         
           ESP32 GND      GPIO38                                                  


                            FLUXOSTATO (opcional)                             
   ╭――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――╮                                                         
   |                                                                    |
   |            Fluxostato                  Resistor Referencia         |                  
   |  3.3V -----/\/\/\/\/\----------------------/\/\/\/\/\-------- GND  |                                                    
   |                                |                                   | 
   |                                |                                   |
   |                                |                                   | 
   |                                |                                   | 
   |                           ADS1115 A1                               |
   ╰――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――╯                                                        


                                 MAX6675 (opcional)
                           ╭――――――――――――――――――――――――――――╮
                           |╭――――╮                      |
    Negativo Termopar <--- || \/ | -   | | | |   SO [■] | ---> ■ GPIO40
                           || /\ |    ╭―――――――╮  CS [■] | ---> ■ GPIO42
                           ||====|◯  |       | SCK [■] | ---> ■ GPIO41
                           || \/ |    ╰―――――――╯ VCC [■] | ---> ■ 3.3V
    Positivo Termopar <--- || /\ | +   | | | |  GND [■] | ---> ■ GND
                           |╰――――╯            =         |
                           ╰――――――――――――――――――――――――――――╯
```

# Funcionamento:

Quando detectado que pressão é inferior ou igual a mínima definida pelo usuário(configuração padrão pre ajustada em 0.5bar), a bomba é ligada e irá parar quando atingir a pressão de parada definida pelo usuário(configuração padrão pré ajustada em 1.2bar).

É totalmente possível usar sem Fluxostato ou MAX6675+Termopar.

# Características:

Os componentes são checados periodicamente, em caso de alguma falha, a bomba/motor não será ligado até que o erro seja corrigido apertando o botão KEY 0 ao lado do encoder.

Caso desejado pode ser monitorado via Home Assistant ou web server.

É possível ir além nas automações se integrado ao Home Assistant.

### Módulo ADS1115:

Monitoramento automático e não pode ser desativado. Caso o modulo venha a falhar, o erro 0x02 será apresentado e a bomba não irá ligar.

### Transdutor:

Monitoramento automático, o erro 0x04 é apresentado caso o valor apresentado pelo adc seja 15% menor que o valor mínimo de calibração.

Exemplo: Definimos 0.44V como 0bar então é permitido um desvio de 15% resultando em 0,374V. Qualquer coisa abaixo disso é considerado como falha no transdutor.

Esse desvio existe para permitir possíveis variações devido o modulo ADS1115 que pode não ser tão preciso.

### Monitorar Falta de Água(Opcional):

Ao ligar o monitor do fluxostato no menu e definir o "Tempo Limite do Fluxostato"(configuração padrão pré ajustada em 15 segundos), A bomba irá ligar e se após 15 segundos o fluxostato não abrir a bomba irá parar. O erro 0x08 será apresentado.

Também será monitorado o fluxostato aberto durante a parada da bomba(impurezas ou desgaste podem danificar a mola do fluxostato). Quando a bomba parar de funcionar e após o "Tempo Limite do Fluxostato"(configuração padrão pre ajustada em 15 segundos) passar e o fluxostato ainda estiver na posição aberta, o erro 0x10 será apresentado.

Observação: Caso a bomba inicie com água e durante a operação faltar, o erro 0x08 será apresentado.

### Tempo Máximo de Funcionamento da Bomba:

É possível definir o tempo máximo de funcionamento da bomba no menu(configuração padrão pré ajustada em 600 segundos = 5 minutos). Se por algum motivo ela funcionar por mais que o tempo definido, o erro 0x01 será apresentado.

A contagem é reiniciada a cada ciclo de sucesso. Novamente: O erro 0x01 só é apresentado em caso do ciclo atual for maior ou igual ao tempo máximo definido.

### Valores de Pressão de Acionamento e Parada:

É impossível o funcionamento caso a configuração de acionamento seja maior ou igual a configuração de parada. O erro 0x80 será mostrado indicando os valores atuais para correção.

Também existe um tolerância mínima de 0.3bar entre acionamento e parada. Caso a tolerância não seja respeita o erro 0x20 será mostrado.

Exemplo: Se o acionamento for definido em 1bar, a parada mínima precisa ser de 1.3bar

Para maior segurança esse valor é definido apenas no código, caso deseje alterar, será necessário compilar novamente.

### Monitorar Temperatura da Bomba(opcional):

Caso a opção de monitorar temperatura seja ativada, a cada 30 segundos a leitura do termopar é atualizada.

É possível definir a temperatura máxima de trabalho da bomba no menu na opção "Temperatura Máx."(configuração padrão pré ajustada em 50°C).

Ao atingir a temperatura máxima definida, a bomba irá parar e apresentará erro 0x40. O funcionamento só retornará caso a temperatura seja 10% inferior ao valor definido.

O Modulo MAX6675 também será monitorado. Em caso de falha no modulo, o erro 0x100 é apresentado.

Caso o Termopar apresente falha, o erro 0x200 é apresentado.

Nota: Lembre-se de colocar o termopar o mais próximo possível do motor/bomba para uma leitura mais precisa.

### Pausar Operação:

Em caso de ausência prologada ou modificações, é recomendado pausar o controlador. Navegue no menu até a opção "Pausar Controlador" e mude o switch para a direita para pausar a operação. Ao ligar o switch "erro" 0x400 será apresentado.

### Erros e possíveis soluções:

| Valor        | Descrição                                        | Solução
| :----------: | :----------------------------------------------: | :--------------------------------------------------------:
| 0x01         | Bomba Excedeu o Tempo Máximo                     | Verifique o tempo máximo definido no menu e possíveis pontos de consumo aberto.
| 0x02         | Falha ADS1115                                    | Módulo ADS1115 precisa se substituído.
| 0x04         | Falha Transdutor                                 | Transdutor precisa se verificado e/ou substituído.
| 0x08         | Falta de Água ou Fluxostato Fechado              | Verifique se a linha contem água e/ou verifique o fluxostato.
| 0x10         | Fluxatato Travado em Aberto                      | Verifique o fluxostato.
| 0x20         | Tole. Min Aci/Des: <=0.3                         | Valor mínimo entre acionamento/parada não respeitado. Ajuste no menu. [1]
| 0x40         | Superaquecimento                                 | Verifique possíveis causas do superaquecimento.
| 0x80         | Press. de Aci. >= Para.                          | Valor de acionamento >= ao valor de parada. Ajuste no menu.
| 0x100        | Falha MAX6675                                    | Módulo MAX6675 precisa se substituído.
| 0x200        | Falha Termopar                                   | Termopar rompido ou precisa ser substituído.
| 0x400        | Controlador Pausado                              | Desligue o switch "Pausar Controlador" no menu.
| 0x800        | X1 == X2(V)                                      | Valor de calibração X1 idêntico ao X2. Verifique a Tensão definida nos menus de calibragem X1 e X2.
| 0x1000       | Y1 == Y2(P)                                      | Valor de calibração Y1 idêntico ao Y2. Verifique a Pressão definida nos menus de calibragem Y1 e Y2.


[1] A diferença mínima entre acionamento e parada precisa ser >= 0.3 bar.

Pode acontecer do erro apresentado não está diretamente relacionado ao componente listado. É recomendado checar falhas em conectores, fios, soldas, resistores e afins.

# Calibração

> [!IMPORTANT]
> É recomendado conferir com um manômetro de confiança na tubulação e se atentar ao valor apresentado no ADC. Os valores abaixo correspondem ao meu setup(150 resistor).

A calibração linear é feita da seguinte maneira:

Com o transdutor na tubulação vazia(ou na tubulação ajustada com menor valor que você quer representar), preencha CALL X1 e CALL Y1.

CALL X1 é o menor valor que você quer representar em V. CALL Y1 é o menor valor que você quer representar em Bar. 

| CALL X1  | CALL Y1 |
| :------: | :-----: |
|   0.58   |    0    |

> [!IMPORTANT]
> No exemplo acima o adc lê 0.44V e isso representa meu valor mínimo desejado que é ZERO BAR. Significa que quando o adc apresentar 0.44V, temos 0 Bar nos tubos.

Com o transdutor na tubulação, pressurize a mesma com o valor desejado e preencha CALL X2 e CALL Y2.

CALL X2 e CALL Y2 é o valor que será calibrado como referencia.

| CALL X2  | CALL Y2 |
| :------- | :-----: |
|   0.85   |    1    |

> [!IMPORTANT]
> No exemplo acima a pressão mostrada pelo manômetro é aproximadamente 1 Bar e o ADC apresenta 0.66V. Isso Significa que quando o ADC apresentar 0.66V, temos 1 Bar nos tubos.

Nota: Para facilitar a calibragem, nas páginas de X1(V) e X2(V) é possível preencher rapidamente com o valor atual do ADC pressionando duas vezes o botão KEY0. Ao preencher com sucesso será mostrado a mensagem "Copiado!" abaixo do titulo. Caso tente preencher enquanto o ADS1115 estiver ausente, com problema ou ausência/falha no transdutor, a mensagem mostrada será "copy ERR" em vermelho abaixo do título.

Mais detalhes sobre a calibração:
https://esphome.io/components/sensor/filter/calibrate_linear/

> [!IMPORTANT]
> Valores de calibragem X1 e X2 não podem ser iguais assim como Y1 e Y2 também não podem ser iguais. Caso aconteça, erros serão apresentados de acordo. Mais informações na parte de "Erros e possíveis soluções".

# Tela inicial

```
          ╭―――――――――――――――――――――――――――――――――――――――――――――――――――╮
     1 -- | °C                    /!\                      \/ | -- 1             
     2 -- | Diagnóstico:                                   OK | -- 2             
     3 -- | Status Bomba:                               (==0) | -- 3             
     4 -- | Pressão:                  1 Bar 14.5 Psi 1.19 Mca | -- 4             
     5 -- | Valor ADC:                                  0.44V | -- 5             
     6 -- | Aci/Para:                     0.50 Bar   1.00 Bar | -- 6             
     7 -- | Tempo de Trabalho:                    0.0s   0.0s | -- 7          
          ╰―――――――――――――――――――――――――――――――――――――――――――――――――――╯


1 - Titulo: Caso o switch de monitorar temperatura esteja ligado, a temperatura atual será mostrada no canto superior esquerdo na cor verde, caso contrario, ele ficará apagado. 
            Se o WIFI estiver ativo, o indicador no canto superior direito ficará verde mostrando que o WIFI está ativo, caso o WIFI seja desconectado ou desligado ele ficará apagado.
            Se o Diagnóstico for diferente de OK, o ícone de warning irá aparecer em vermelho no meio, caso contrário, ficará apagado.

2 - Diagnóstico: Aqui será mostrado possíveis falhas. Caso tudo esteja funcional "OK" é mostrado.

3 - Status Bomba: O switch mostra o funcionamento da bomba.

4 - Pressão: Aqui é mostrado a pressão atual em Bar|Psi|Mca. Caso o switch MODO: esteja em AR, a pressão será mostrada em Bar|Psi|Kgf/cm2

5 - Mostra o valor atual do ADS1115.

6 - Mostra o valor de Acionamento e Parada.

7 - Mostra o tempo de trabalho total e o tempo de duração do último ciclo.
    |
    ╰> Nota: Essa informação é reiniciada toda vez que o controlador é reiniciado.
```

# NAVEGAÇÃO

## Em caso de Falha:

Ao pressionar o botão KEY0(esquerda do encoder) por 2 segundos, o script que checa se as falhas foram corrigidas será executado. Caso tudo esteja OK, o controlador retornará ao funcionamento normal.

*Essa função não limpa o "erro" do controlador pausado, é preciso ir no menu e mudar a posição do switch.

*Ao pressionar por 2 segundos um indicador de checagem de erros deverá aparecer na tela.

## Menu:

Gire o Encoder para <esquerda/direita> para navegar pelo menu. Também é possível navegar na página principal(apenas se a quantidade falhas exceder o que cabe na tela).

Shortpress = Pressione o encoder e solte.

Longpress = Pressione e segure o encoder por um curto período.

Use Longpress para fechar o menu e retornar a tela inicial. Caso o menu fique aberto, quando a tela apagar ele irá fechar.

## Switch: 

```
Gire o Encoder para <esquerda/direita> para Ligar/Desligar o switch selecionado.
Shortpress ou Longpress para sair do Switch.

       ╭―――――――――――――――――――――╮
       |        Switch       |
       |                     |
       |      ╭―――――╭――╮     |
       |  OFF |     |  | ON  | 
       |      ╰―――――╰――╯     |
       |                     |
       ╰―――――――――――――――――――――╯
```

## Spinbox:

```
Shortpress para se mover pelos dígitos. 
Gire o Encoder para <esquerda/direita> para Aumentar/Diminuir o digito selecionado.
Longpress para sair do Spinbox.

       ╭―――――――――――――――――――――――――――――╮
       |          Spinbox            |
       |                             |
       |          ╭――――――╮           |
       |   MIN    |[0]000|   MAX     |
       | 0.00 val ╰――――――╯ 0.00 val  |
       |                             |
       ╰―――――――――――――――――――――――――――――╯

Os spinbox de calibração mostram o valor do ADC na parte inferior.    

       ╭―――――――――――――――――――――――――――――╮
       |          Spinbox            |
       |                             |
       |          ╭――――――╮           |
       |   MIN    |[0]000|   MAX     |
       | 0.00 val ╰――――――╯ 0.00 val  |
       |                             |
       |         ADC: 0.00V          |
       ╰―――――――――――――――――――――――――――――╯

```

## Msgbox:    

```
Gire o Encoder para <esquerda/direita> para navegar entre os botões de Confirmar/Cancelar.
Shortpress no botão selecionado para executar a ação.
Também é possível usar Longpress para sair imediatamente da Msgbox.

       ╭―――――――――――――――――――――――――――――――――╮
       |             MENSAGEM            |
       |                                 |
       | ╭―――――――――――╮    ╭―――――――――――╮  |
       | | Confirmar |    |  Cancelar |  |
       | ╰―――――――――――╯    ╰―――――――――――╯  |
       ╰―――――――――――――――――――――――――――――――――╯         
```

## Slider:    

```
Gire o Encoder para <esquerda/direita> para Diminuir ou Aumentar barra.
Shortpress ou Longpress para sair do Slider. 

       ╭――――――――――――――――――――╮
       |       Slider       |
       |           ╭―╮      |
       |   ╭―――――――| |―╮    |
       |   ╰―――――――| |―╯    | 
       |           ╰―╯      |
       |        80%         |
       ╰――――――――――――――――――――╯         
```

## QRCODE:    

```
Gire o Encoder para <esquerda/direita> ou Shortpress/Longpress para sair do QRCODE.

       ╭――――――――――――――――╮
       |     QRCODE     |
       | ■■ ■   ■■■  ■■ |
       | ■■■■ ■■  ■■ ■■ |
       | ■■ ■ ■■ ■■■ ■  |
       | ■■■■  ■■  ■■■■ |
       ╰――――――――――――――――╯         
```

# MENU PRINCIPAL

```
            ╭――――――――――――――――――――――――――╮
            |           MENU           |
            |                          |
       1 -- |     Mnt. Fluxostato      | -- 1 
       2 -- |     Mnt. Temperatura     | -- 2 
       3 -- |     Pressão Aci.         | -- 3
       4 -- |     Pressão Para.        | -- 4 
       5 -- |     Temperatura Máx.     | -- 5
       6 -- |     Tempo Máx. Bomba     | -- 6
       7 -- |     Tempo Lim. Flux      | -- 7
       8 -- |     Cal X1(V)            | -- 8
       9 -- |     Cal Y1(P)            | -- 9
      10 -- |     Cal X2(V)            | -- 10
      11 -- |     Cal Y2(P)            | -- 11
      12 -- |     Teste Relay          | -- 12
      13 -- |     WIFI                 | -- 13 
      14 -- |     Modo Opr.            | -- 14
      15 -- |     Pausar Controlador   | -- 14
      16 -- |     Brilho do Disp.      | -- 14
      17 -- |     Manual               | -- 15
      18 -- |     Reiniciar            | -- 18
      19 -- |     Restaurar Cfg        | -- 19
            ╰――――――――――――――――――――――――――╯ 


1 - Mon. Fluxostato (Switch) - Ative caso deseje monitorar possiveis faltas de água.

2 - Mon. Temperatura  (Switch) - Ative caso deseje monitorar a temperatura do termopar.

3 - Pressão Aci. (Spinbox) - Ajuste a pressão de acionamento desejada.

4 - Pressão Para. (Spinbox) - Ajuste a pressão de parada desejada.

5 - Temperatura Máx. (Spinbox) - Ajuste de temperatura máxima permitida durante o funcionamento. 
       |
       ╰> Essa opção só é exibida caso o switch Mon. Temperatura esteja ligado.

6 - Tempo Máx. Bomba (Spinbox) - Ajuste o tempo máximo permitido para cada ciclo da bomba.

7 - Tempo Lim. Flux (Spinbox) - Ajuste o tempo máximo que o fluxostato deve reagir a falta de água e na parada da bomba.

8 - Cal X1(V) (Spinbox) - Ajusta o valor X1. Mais informações na parte de calibração.

9 - Cal Y1(P) (Spinbox) - Ajusta o valor Y1. Mais informações na parte de calibração.

10 - Cal X2(V) (Spinbox) - Ajusta o valor X2. Mais informações na parte de calibração.

11 - Cal Y2(P) (Spinbox) - Ajusta o valor Y2. Mais informações na parte de calibração.

12 - Teste Relay (Switch) - Ao ativar inicia um teste que liga relay por 5 segundos e logo em seguida desliga.

13 - WIFI (Switch) - Ative caso deseje ligar o WIFI.
       |
       ╰> Para navegar até o webserver, digite pressureswitchs3.local no seu navegador. Caso a url não funcione, encontre o ip do controlador e cole no seu navegador.
       |
       ╰> Caso queira integrar no Home Assistant, use o mesmo endereço de IP no dashboard.

14 - Modo Opr. (Switch) - Ao posicionar na esquerda o modo ÁGUA é ativado, na direita o modo AR é ativado.
       |
       ╰> MODO: ÁGUA - Ao colocar nesse modo no display é mostrado os valores de pressão em Bar|Psi|Mca e a opção de monitorar fluxostato é mostrada no menu.
       |
       ╰> MODO: AR - Ao colocar nesse modo no display é mostrado os valores de pressão em Bar|Psi|kgf/cm2. A opção de monitorar o fluxostato não será mostrada no menu e caso switch do fluxostato esteja ativo, será desligado. Na página principal, o texto "Pressão:" vira "P:" acomodar a quantidade de valores mostrados.     

15 - Pausar Controlador (Switch) - Ao Ativar pausa a operação do controlador. Ideal para pausar a operação em caso de ausência prolongada.   
       
16 - Brilho do Disp. (Slider) - Ajusta o brilho do display.

17 - Manual (QRCODE) - Use para ter acesso ao Manual(este documento).

18 - Reiniciar (Msgbox) - Usado para reiniciar o controlador.

19 - Restaurar Cfg (Msgbox) - Usado para restaurar as configurações do controlador.

Nota: Caso o Menu esteja ativo e alguma configuração seja modificada via web_server, talvez seja necessário reabrir o menu para refletir as alterações no display.
```

# Avançado:

Não é recomendado usar sem display mas é totalmente possível aceitando as limitações do webserver. Caso queira usar sem display, modifique conforme desejado.

Vantagens do display:

```
Informações de fácil acesso e navegação sem dependendencia de WIFI/webserver/rest api;
Conversões de pressão em Psi, Mca e kgf/cm2;
Ajuste de Brilho;
QRCODE para o manual;
```

*Ao remover o display lembre-se de modificar o WIFI para enable_on_boot: true

# Agradecimentos:
ESPHome - https://esphome.io

Ideia de Menu e Navegação - https://github.com/RealDeco/SendspinZero

Dynamic timer - https://karlquinsland.com/esphome-dynamic-timer/
