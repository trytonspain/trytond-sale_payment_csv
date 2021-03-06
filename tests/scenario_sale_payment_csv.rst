=============
Sale Scenario
=============

Imports::

    >>> import datetime
    >>> from dateutil.relativedelta import relativedelta
    >>> from decimal import Decimal
    >>> from operator import attrgetter
    >>> from os import path
    >>> from proteus import config, Model, Wizard
    >>> from trytond.modules.company.tests.tools import create_company, \
    ...     get_company
    >>> from trytond.modules.account.tests.tools import create_fiscalyear, \
    ...     create_chart, get_accounts, create_tax
    >>> from.trytond.modules.account_invoice.tests.tools import \
    ...     set_fiscalyear_invoice_sequences, create_payment_term
    >>> today = datetime.date.today()

Create database::

    >>> config = config.set_trytond()
    >>> config.pool.test = True

Install sale_payment_csv::

    >>> Module = Model.get('ir.module')
    >>> sale_payment_csv_module, = Module.find([
    ...     ('name', '=', 'sale_payment_csv')
    ...     ])
    >>> Module.install([sale_payment_csv_module.id], config.context)
    >>> Wizard('ir.module.install_upgrade').execute('upgrade')

Create company::

    >>> _ = create_company()
    >>> company = get_company()
    >>> currency = company.currency

Reload the context::

    >>> User = Model.get('res.user')
    >>> Group = Model.get('res.group')
    >>> config._context = User.get_preferences(True, config.context)

Create sale user::

    >>> sale_user = User()
    >>> sale_user.name = 'Sale'
    >>> sale_user.login = 'sale'
    >>> sale_user.main_company = company
    >>> sale_group, = Group.find([('name', '=', 'Sales')])
    >>> sale_user.groups.append(sale_group)
    >>> sale_user.save()

Create stock user::

    >>> stock_user = User()
    >>> stock_user.name = 'Stock'
    >>> stock_user.login = 'stock'
    >>> stock_user.main_company = company
    >>> stock_group, = Group.find([('name', '=', 'Stock')])
    >>> stock_user.groups.append(stock_group)
    >>> stock_user.save()

Create account user::

    >>> account_user = User()
    >>> account_user.name = 'Account'
    >>> account_user.login = 'account'
    >>> account_user.main_company = company
    >>> account_group, = Group.find([('name', '=', 'Account')])
    >>> account_user.groups.append(account_group)
    >>> account_user.save()

Create fiscal year::

    >>> fiscalyear = set_fiscalyear_invoice_sequences(
    ...     create_fiscalyear(company))
    >>> fiscalyear.click('create_period')

Create chart of accounts::

    >>> _ = create_chart(company)
    >>> accounts = get_accounts(company)
    >>> receivable = accounts['receivable']
    >>> revenue = accounts['revenue']
    >>> expense = accounts['expense']
    >>> cash = accounts['cash']

    >>> Journal = Model.get('account.journal')
    >>> Sequence = Model.get('ir.sequence')
    >>> StatementJournal = Model.get('account.statement.journal')
    >>> acc_journl_seq, = Sequence.find([('code', '=', 'account.journal')])
    >>> journal = Journal()
    >>> journal.name = 'Journal'
    >>> journal.type = 'statement'
    >>> journal.sequence = acc_journl_seq
    >>> journal.credit_account = cash
    >>> journal.debit_account = cash
    >>> journal.save()
    >>> write_off_journal = Journal()
    >>> write_off_journal.name = 'Write Off Journal'
    >>> write_off_journal.type = 'statement'
    >>> write_off_journal.sequence = acc_journl_seq
    >>> write_off_journal.credit_account = cash
    >>> write_off_journal.debit_account = cash
    >>> write_off_journal.save()
    >>> statement_journal = StatementJournal()
    >>> statement_journal.name = 'Statement Journal'
    >>> statement_journal.journal = journal
    >>> statement_journal.currency = currency
    >>> statement_journal.company = company
    >>> statement_journal.validation = 'balance'
    >>> statement_journal.save()
    >>> write_off_statement_journal = StatementJournal()
    >>> write_off_statement_journal.name = 'Write Off Statement Journal'
    >>> write_off_statement_journal.journal = write_off_journal
    >>> write_off_statement_journal.currency = currency
    >>> write_off_statement_journal.company = company
    >>> write_off_statement_journal.validation = 'balance'
    >>> write_off_statement_journal.save()

Create parties::

    >>> Party = Model.get('party.party')
    >>> supplier = Party(name='Supplier')
    >>> supplier.save()
    >>> customer = Party(name='Customer')
    >>> customer.save()

Create payment term::

    >>> payment_term = create_payment_term()
    >>> payment_term.save()

Create Product Price List::

    >>> ProductPriceList = Model.get('product.price_list')
    >>> product_price_list = ProductPriceList()
    >>> product_price_list.name = 'Price List'
    >>> product_price_list.company = company
    >>> product_price_list.save()

Create Sale Shop::

    >>> Shop = Model.get('sale.shop')
    >>> shop = Shop()
    >>> shop.name = 'Sale Shop'
    >>> Location = Model.get('stock.location')
    >>> warehouse, = Location.find([
    ...         ('type', '=', 'warehouse'),
    ...         ])
    >>> shop.warehouse = warehouse
    >>> shop.price_list = product_price_list
    >>> shop.payment_term = payment_term
    >>> sequence, = Sequence.find([
    ...         ('code', '=', 'sale.sale'),
    ...         ])
    >>> shop.sale_sequence = sequence
    >>> shop.sale_invoice_method = 'shipment'
    >>> shop.sale_shipment_method = 'order'
    >>> shop.save()
    >>> sale_user.shops.append(shop)
    >>> sale_user.shop = shop
    >>> sale_user.save()

Create category::

    >>> ProductCategory = Model.get('product.category')
    >>> category = ProductCategory(name='Category')
    >>> category.save()

Create product::

    >>> ProductUom = Model.get('product.uom')
    >>> unit, = ProductUom.find([('name', '=', 'Unit')])
    >>> ProductTemplate = Model.get('product.template')
    >>> Product = Model.get('product.product')
    >>> product = Product()
    >>> template = ProductTemplate()
    >>> template.name = 'product'
    >>> template.category = category
    >>> template.default_uom = unit
    >>> template.type = 'goods'
    >>> template.purchasable = True
    >>> template.salable = True
    >>> template.list_price = Decimal('10')
    >>> template.cost_price = Decimal('5')
    >>> template.cost_price_method = 'fixed'
    >>> template.account_expense = expense
    >>> template.account_revenue = revenue
    >>> template.save()
    >>> product.template = template
    >>> product.save()

Create payment term::

    >>> payment_term = create_payment_term()
    >>> payment_term.save()

Create an Inventory::

    >>> config.user = stock_user.id
    >>> Inventory = Model.get('stock.inventory')
    >>> InventoryLine = Model.get('stock.inventory.line')
    >>> Location = Model.get('stock.location')
    >>> storage, = Location.find([
    ...         ('code', '=', 'STO'),
    ...         ])
    >>> inventory = Inventory()
    >>> inventory.location = storage
    >>> inventory.save()
    >>> inventory_line = InventoryLine(product=product, inventory=inventory)
    >>> inventory_line.quantity = 100.0
    >>> inventory_line.expected_quantity = 0.0
    >>> inventory.save()
    >>> inventory_line.save()
    >>> Inventory.confirm([inventory.id], config.context)
    >>> inventory.state
    u'done'

Make 5 sales::

    >>> config.user = sale_user.id
    >>> Sale = Model.get('sale.sale')
    >>> SaleLine = Model.get('sale.line')
    >>> quantities = [1.0, 2.0, 3.0, 4.0, 5.0]
    >>> for quantity in quantities:
    ...     sale = Sale()
    ...     sale.party = customer
    ...     sale.shop = shop
    ...     sale.payment_term = payment_term
    ...     sale.invoice_method = 'order'
    ...     sale_line = SaleLine()
    ...     sale.lines.append(sale_line)
    ...     sale_line.product = product
    ...     sale_line.quantity = quantity
    ...     sale.save()
    ...     Sale.quote([sale.id], config.context)
    ...     Sale.confirm([sale.id], config.context)
    ...     Sale.process([sale.id], config.context)

Create account statement csv profile with four lines::

    >>> ProfileCSV = Model.get('profile.csv')
    >>> ProfileCSVColumn = Model.get('profile.csv.column')
    >>> ModelModel = Model.get('ir.model')
    >>> ModelField = Model.get('ir.model.field')
    >>> account_statement_line_model, = ModelModel.find([
    ...         ('model', '=', 'account.statement.line')
    ...         ])
    >>> profile = ProfileCSV()
    >>> profile.name = 'Test profile'
    >>> profile.model = account_statement_line_model
    >>> profile.separator = ','
    >>> profile.quote = '"'
    >>> profile.match_expression = 'row[5] != "Completado" or row[31] == ""'
    >>> profile.active = True
    >>> profile.thousands_separator = 'none'
    >>> profile.decimal_separator = ','
    >>> profile.journal = statement_journal
    >>> profile.character_encoding = 'utf-8'
    >>> profile.write_off_journal = write_off_statement_journal
    >>> profile.sale_domain = "[('reference', '=', row[31])]"
    >>> profile.sale_state = "[('state', 'not in', ['cancel', 'done']), "
    >>> profile.sale_state += "('invoice_state', '!=', 'paid')]"
    >>> profile.sale_amount = "[('total_amount_cache', '>', values['amount'] *"
    >>> profile.sale_amount += " Decimal(0.99)), ('total_amount_cache', '<', "
    >>> profile.sale_amount += "values['amount'] * Decimal(1.01))]"
    >>> profile_column = ProfileCSVColumn()
    >>> profile.columns.append(profile_column)
    >>> profile_column.column = '31'
    >>> model_field, = ModelField.find([
    ...         ('name', '=', 'description'),
    ...         ('model', '=', account_statement_line_model.id),
    ...         ])
    >>> profile_column.field = model_field
    >>> profile_column.add_to_domain = True
    >>> profile_column = ProfileCSVColumn()
    >>> profile.columns.append(profile_column)
    >>> profile_column.column = '7'
    >>> model_field, = ModelField.find([
    ...         ('name', '=', 'amount'),
    ...         ('model', '=', account_statement_line_model.id),
    ...         ])
    >>> profile_column.field = model_field
    >>> profile_column.add_to_domain = True
    >>> profile.save()

Import CSV Payment Wizard with file of 5 statements::

    >>> import_csv_payment = Wizard('import.csv.payment.from.sale')
    >>> import_csv_payment.form.attach = False
    >>> csv_file = (path.dirname(path.realpath(__file__)) + '/paypal.csv')
    >>> csv_file_content = open(csv_file, 'rb')
    >>> import_csv_payment.form.import_file = csv_file_content.read()
    >>> import_csv_payment.form.profile = profile
    >>> import_csv_payment.execute('import_file')

Check created statements

    >>> Statement = Model.get('account.statement')
    >>> StatementLine = Model.get('account.statement.line')
    >>> statement, = Statement.find([
    ...     ('journal', '=', statement_journal.id),
    ...     ])
    >>> statement.state
    u'draft'
    >>> statement_lines = StatementLine.find([
    ...     ('statement', '=', statement.id),
    ...     ])
    >>> len(statement_lines)
    2
    >>> statement_line_1 = statement_lines[0]
    >>> statement_line_2 = statement_lines[1]
    >>> sale_line_1 = statement_line_1.sale.lines[0]
    >>> sale_line_1.quantity
    1.0
    >>> sale_line_1.amount - statement_line_1.amount
    Decimal('0.00')
    >>> sale_line_2 = statement_line_2.sale.lines[0]
    >>> sale_line_2.quantity
    2.0
    >>> sale_line_2.amount - statement_line_2.amount
    Decimal('0.01')
    >>> write_off_statement, = Statement.find([
    ...     ('journal', '=', write_off_statement_journal.id),
    ...     ])
    >>> write_off_statement.state
    u'draft'
    >>> write_off_statement_lines = StatementLine.find([
    ...     ('statement', '=', write_off_statement.id),
    ...     ])
    >>> len(write_off_statement_lines)
    1
    >>> write_off_statement_line, = write_off_statement_lines
    >>> sale_line_2.amount - write_off_statement_line.amount
    Decimal('19.99')
