.. include:: <s5defs.txt>

.. container:: center, huge

    Hackeando Emuladores

    por diversão e conhecimento

.. container:: center, small

    Henrique Bastos


Quem?
-----

.. class:: incremental

* `@henriquebastos <http://twitter.com/henriquebastos>`_
* henrique@bastos.net
* http://henriquebastos.net
* http://welcometothedjango.com.br


Quem?
-----

.. container:: center, huge

    .. image:: assets/loogica_logo.svg
        :height: 250px
        :width: 600 px

.. container:: center, huge

    .. image:: assets/dekode_logo.svg
        :height: 250px
        :width: 600 px

O que é Chip8
-------------

* Linguagem de programação
* Interpretada
* Minimalista
* Joseph Weisbecker
* 1970

Características
---------------

* 31 instruções hexadecimal de 4 bytes
* Sem processamento de texto
* Áudio primitivo
* Vídeo primitivo
* Keyboard
* Boa para máquinas com memória escassa

Arquitetura
-----------

.. container:: tiny

    .. sourcecode:: python

        class TestChip8Architecture(TestCase):
            def setUp(self):
                self.cpu = Chip8()

            def test_memory_length(self):
                'Chip8 has 4096 bytes of memory.'
                self.assertEqual(4096, len(self.cpu.memory))

            def test_register_count(self):
                'Chip8 has 16 registers.'
                self.assertEqual(16, len(self.cpu.registers))

            def test_program_counter(self):
                'Chip8 has a program counter starting at 0x200.'
                self.assertEqual(0x200, self.cpu.program_counter)

            def test_stack(self):
                'Chip8 has a stack to return from jumps and calls.'
                self.assertIsInstance(self.cpu.stack, list)

Arquitetura
-----------

.. container:: tiny

    .. sourcecode:: python

        class TestChip8Architecture(TestCase):
            ''' continue... '''

            def test_screen(self):
                'Chip8 has a 64 * 32 screen (2048 pixels).'
                self.assertEqual(2048, len(self.cpu.screen))

            def test_index_register(self):
                'Chip8 has an index register.'
                self.assertEqual(0, self.cpu.index_register)

            def test_keyboard(self):
                'Chip8 has a hex keyboard with 16 keys.'
                self.assertEqual(16, len(self.cpu.keyboard))

            def test_delay_timer(self):
                'Chip8 has a delay timer that counts to 0 at 60Hz.'
                self.assertEqual(0, self.cpu.delay_timer)

            def test_sound_timer(self):
                'Chip8 has a count down sound timer that beeps on non-zero value.'
                self.assertEqual(0, self.cpu.sound_timer)

Instruções Constantes
---------------------

``00E0``: Clear the screen

::

    Hex   Bin
    00E0  0000 0000 1110 0000

Instruções com endereços
------------------------

``1NNN``: Jump to address NNN

::

    Hex   Bin
    1200  0001 0010 0000 0000

Com 1 registrador e 1 valor
---------------------------

``6XNN``: Store number NN in register VX

::

    Hex   Bin
    6A00  0110 1010 0000 0000

Com 2 registradores
-------------------

``8XY1``: Set VX to VX OR VY

::

    Hex   Bin
    80F1  1000 0000 1111 0001

Com 2 registradores e 1 valor
-----------------------------

``DXYN``: Draw a sprite at position VX, VY with N bytes...

::

    Hex   Bin
    D011  1101 0000 0001 0001


Main loop
---------

.. sourcecode:: python

    class Chip8(object):
        def cycle(self):
            word = self.fetch()
            instruction, args = self.decode(word)
            self.execute(instruction, args)
            self.decrement_timers()

Multiplexador
-------------

Decodifica instruções e seus parâmetros com `bitmask <https://en.wikipedia.org/wiki/Mask_(computing)>`_

.. container:: tiny

    .. sourcecode:: python

        class TestOpcodeDecode(TestCase):
            def setUp(self):
                self.cpu = Chip8()

            def test_00E0(self):
                self.assertEqual(self.cpu.decode(0x00E0), (0x00E0, tuple()))

            def test_1NNN(self):
                self.assertEqual(self.cpu.decode(0x1200), (0x1, (0x200,)))

            def test_6XNN(self):
                self.assertEqual(self.cpu.decode(0x6199), (0x6, (0x1, 0x99)))

            def test_8XY4(self):
                self.assertEqual(self.cpu.decode(0x8124), (0x8004, (0x1, 0x2)))

            def test_DXYN(self):
                self.assertEqual(self.cpu.decode(0xD205), (0xD, (0x2, 0x0, 0x5)))



Instruções de Dados
-------------------

.. container:: small

    ::

        6XNN Store number NN in register VX
        8XY0 Store the value of register VY in register VX

        7XNN Add the value NN to register VX

        8XY4 Add the value of register VY to register VX; VF=01 if carry.
        8XY5 Subtract the value of register VY from register VX; VF=0 if borrow.
        8XY7 Set register VX to the value of VY minus VX; VF=0 if borrow.

        8XY2 Set VX to VX AND VY
        8XY1 Set VX to VX OR VY
        8XY3 Set VX to VX XOR VY

        8XY6 Store the value of register VY shifted right one bit in register VX; VF=LSB
        8XYE Store the value of register VY shifted left one bit in register VX; VF=MSB

        CXNN Set VX to a random number with a mask of NN

Instruções de Dados: Exemplo
----------------------------

.. container:: tiny

    .. sourcecode:: python

        # Spec
        def test_8XY4(self):
            self.registers(V1=0xF, V2=0xF0)
            self.execute(0x8124)
            self.assertEqual(self.cpu.registers[1], 0xFF)
            self.assertEqual(self.cpu.registers[0xF], 0x0)
            self.assertEqual(self.cpu.program_counter, 0x202)

        def test_8XY4_overflow(self):
            self.registers(V1=0x1, V2=0xFF)
            self.execute(0x8124)
            self.assertEqual(self.cpu.registers[1], 0x0)
            self.assertEqual(self.cpu.registers[0xF], 0x1)
            self.assertEqual(self.cpu.program_counter, 0x202)

        # Impl
        class Chip8(object):
            def op_8XY4(self, X, Y):
                '''Add the value of register VY to register VX
                   Set VF to 01 if a carry occurs
                   Set VF to 00 if a carry does not occur'''
                value = self.registers[X] + self.registers[Y]
                self.registers[X] = byte(value)
                self.registers[0xF] = 1 if value > 0xFF else 0
                self.increment_program_counter()

Fluxo de Controle
-----------------

::

    1NNN Jump to address NNN
    BNNN Jump to address NNN + V0

Fluxo de Controle: Exemplo
--------------------------

.. container:: tiny

    .. sourcecode:: python

        # Spec
        def test_1NNN(self):
            'Jump to address NNN'
            self.execute(0x1400)
            self.assertEqual(0x400, self.cpu.program_counter)

        def test_BNNN(self):
            'Jump to address NNN + V0'
            self.registers(V0=0x2)
            self.execute(0xB400)
            self.assertEqual(self.cpu.program_counter, 0x402)

        # Impl
        class Chip8(object):
            def op_1NNN(self, address):
                'Jump to address NNN.'
                self.program_counter = address

            def op_BNNN(self, address):
                'Jump to address NNN + V0.'
                self.program_counter = address + self.registers[0]

Subrotinas
----------

::

    2NNN Execute subroutine starting at address NNN
    00EE Return from a subroutine

Subrotinas: Exemplo
-------------------

.. container:: tiny

    .. sourcecode:: python

        # Spec
        def test_2NNN(self):
            'Execute subroutine starting at address NNN'
            self.execute(0x2400)
            self.assertEqual(0x400, self.cpu.program_counter)
            self.assertSequenceEqual([0x202], self.cpu.stack)

        def test_00EE(self):
            'Return from a subroutine'
            self.cpu.stack.append(0x200)
            self.execute(0x00EE, at=0x400)
            self.assertSequenceEqual([], self.cpu.stack)
            self.assertEqual(0x200, self.cpu.program_counter)

        # Impl
        class Chip8(object):
            def op_2NNN(self, address):
                'Execute subroutine starting at address NNN.'
                self.increment_program_counter()
                self.stack.append(self.program_counter)
                self.program_counter = address

            def op_00EE(self):
                'Return from a subroutine.'
                self.program_counter = self.stack.pop()

Jumps Condicionais
------------------

.. container:: small

    ::

        3XNN Skip next if the value of register VX equals NN

        5XY0 Skip next if the value of register VX
             is equal to the value of register VY

        4XNN Skip next if the value of register VX
             is not equal to NN

        9XY0 Skip next if the value of register VX
             is not equal to the value of register VY

Jumps Condicionais: Exemplo
-------------------------------------

Skip next if the value of register *VX* != *NN*

.. container:: tiny

    .. sourcecode:: python

        # Spec
        def test_3XNN_equality(self):
            self.registers(V5=0x42)
            self.execute(0x3542)
            self.assertEqual(0x204, self.cpu.program_counter)

        def test_3XNN_inequality(self):

            self.registers(V5=0x42)
            self.execute(0x3524)
            self.assertEqual(0x202, self.cpu.program_counter)

        # Impl
        class Chip8(object):
            def op_3XNN(self, X, NN):
                'Skip next if the value of VX == NN.'
                if self.registers[X] == NN:
                    self.skip_next_instruction()
                else:
                    self.increment_program_counter()

Timers
------

.. container:: small

    ::

        FX15 Set the delay timer to the value of register VX
        FX07 Store the current value of the delay timer in register VX
        FX18 Set the sound timer to the value of register VX

Input
-----

16 teclas: 0 à F

.. container:: small

    ::

        FX0A Wait for a keypress and store the result in register VX

        EX9E Skip next if the key corresponding to the hex value
             currently stored in register VX is pressed

        EXA1 Skip next if the key corresponding to the hex value
             currently stored in register VX is not pressed

Gráficos
--------

* XOR Mode
* Sprites: 8 pixels de largura por 1 à 15 pixels de altura.

Registrador I
~~~~~~~~~~~~~

.. container:: small

    ::

        ANNN Store memory address NNN in register I
        FX1E Add the value stored in register VX to register I

Desenho
~~~~~~~

.. container:: small

    ::

        DXYN Draw a sprite at position VX, VY with N bytes of sprite data
             starting at the address stored in I.

        Set VF to 01 if any set pixels are changed to unset, and 00 otherwise

Fontes
------

* De 0 à F

.. container:: small

    ::

        0              1              2              3

        Hex Bin        Hex Bin        Hex Bin        Hex Bin

        F0  ####0000   20  00#00000   F0  ####0000   F0  ####0000
        90  #00#0000   60  0##00000   10  000#0000   10  000#0000
        90  #00#0000   20  00#00000   F0  ####0000   F0  ####0000
        90  #00#0000   20  00#00000   80  #0000000   10  000#0000
        F0  ####0000   70  0###0000   F0  ####0000   F0  ####0000

Gráficos: Exemplo
-----------------

.. container:: tiny

    .. sourcecode:: python

        # Spec
        def test_DXYN_no_collision(self):
            self.cpu.index_register = FONT_ADDRESS
            self.registers(V0=0, V1=0)
            self.execute(0xD005)
            self.assertDrawn(self.cpu.memory.font(0), 0, 1)
            self.assertFalse(self.collision())
            self.assertEqual(self.cpu.program_counter, 0x202)

        def test_DXYN_with_collision(self):
            sprite = [0xF0, 0x90, 0x90, 0x90, 0xF0]
            self.cpu.memory.load(0, sprite)
            self.cpu.screen.draw([0xFF], 0, 0) # previous state
            self.cpu.index_register = 0
            self.registers(V0=0, V1=0)
            self.execute(0xD005)
            self.assertDrawn([0x0F, 0x90, 0x90, 0x90, 0xF0], 0, 1)
            self.assertTrue(self.collision())
            self.assertEqual(self.cpu.program_counter, 0x202)

        # Impl
        class Chip8(object):
            def op_DXYN(self, VX, VY, N):
                x = self.registers[VX]
                y = self.registers[VY]
                sprite = self.memory.read(self.index_register, N)
                collision = self.screen.draw(sprite, x, y)
                self.registers[0xF] = bool(collision)
                self.increment_program_counter()

Outros
------

.. container:: center

    Tem mais, mas não dá tempo! :)

PyGame
------

.. container:: tiny

    .. sourcecode:: python

        screen = pygame.display.set_mode((640, 320))
        background = pygame.Surface(screen.get_size())
        background = background.convert()

        emulator = Chip8()

        # Lê a ROM...
        with open(args.rom, 'rb') as f:
            buffer = bytearray(4092 - 0x200)
            f.readinto(buffer)
            emulator.memory.load(0x200, buffer)

        while True:
            for event in pygame.event.get():
                if event.type == KEYDOWN:
                    key = KEYMAP.get(event.key, 0)
                    if key:
                        emulator.keyboard[key] = 1
                elif event.type == KEYUP:
                    key = KEYMAP.get(event.key)
                    if key:
                        emulator.keyboard[key] = 0

            emulator.cycle()
            # Atualiza tela
            background.fill((0, 0, 0))
            for x in range(64):
                for y in range(32):
                    if emulator.screen.get(x, y):
                        pixel = Rect(x*10, y*10, 10, 10)
                        pygame.draw.rect(background, (255, 255, 255), pixel)
            screen.blit(background, background.get_rect())
            pygame.display.update()



Evolução?
---------

* Implementar S-Chip8
* Disassembler
* Macro Assembler
* ...

Obrigado
--------

* `@henriquebastos <http://twitter.com/henriquebastos>`_
* henrique@bastos.net
* http://henriquebastos.net
* http://welcometothedjango.com.br
