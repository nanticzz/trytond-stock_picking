======================
Stock Picking Scenario
======================

Imports::

    >>> import datetime
    >>> from dateutil.relativedelta import relativedelta
    >>> from decimal import Decimal
    >>> from proteus import config, Model, Wizard
    >>> from trytond.tests.tools import activate_modules
    >>> from trytond.modules.company.tests.tools import create_company, \
    ...     get_company
    >>> from trytond.modules.account.tests.tools import create_fiscalyear, \
    ...     create_chart, get_accounts, create_tax
    >>> from trytond.modules.account_invoice.tests.tools import \
    ...     set_fiscalyear_invoice_sequences, create_payment_term
    >>> today = datetime.date.today()
    >>> yesterday = today - relativedelta(days=1)

Install stock Module::

    >>> config = activate_modules(['sale', 'stock_picking'])

Create company::

    >>> _ = create_company()
    >>> company = get_company()

Create fiscal year::

    >>> fiscalyear = set_fiscalyear_invoice_sequences(
    ...     create_fiscalyear(company))
    >>> fiscalyear.click('create_period')

Create chart of accounts::

    >>> _ = create_chart(company)
    >>> accounts = get_accounts(company)
    >>> revenue = accounts['revenue']
    >>> expense = accounts['expense']
    >>> cash = accounts['cash']

Create payment term::

    >>> payment_term = create_payment_term()
    >>> payment_term.save()

Create customer::

    >>> Party = Model.get('party.party')
    >>> customer = Party(name='Customer')
    >>> customer.save()

Create tax::

    >>> tax = create_tax(Decimal('.10'))
    >>> tax.save()

Create account categories::

    >>> ProductCategory = Model.get('product.category')
    >>> account_category = ProductCategory(name="Account Category")
    >>> account_category.accounting = True
    >>> account_category.account_expense = expense
    >>> account_category.account_revenue = revenue
    >>> account_category.save()

    >>> account_category_tax, = account_category.duplicate()
    >>> account_category_tax.customer_taxes.append(tax)
    >>> account_category_tax.save()

Create products::

    >>> ProductUom = Model.get('product.uom')
    >>> ProductTemplate = Model.get('product.template')
    >>> Product = Model.get('product.product')
    >>> unit, = ProductUom.find([('name', '=', 'Unit')])
    >>> template = ProductTemplate()
    >>> template.name = 'Product'
    >>> template.default_uom = unit
    >>> template.type = 'goods'
    >>> template.list_price = Decimal('20')
    >>> template.salable = True
    >>> template.account_category = account_category_tax
    >>> template.save()
    >>> product1, = template.products
    >>> product1.code = 'P1'
    >>> product1.save()

    >>> template = ProductTemplate()
    >>> template.name = 'Product'
    >>> template.default_uom = unit
    >>> template.type = 'goods'
    >>> template.list_price = Decimal('20')
    >>> template.salable = True
    >>> template.account_category = account_category_tax
    >>> template.save()
    >>> product2, = template.products
    >>> product2.code = 'P2'
    >>> product2.save()

Get stock locations::

    >>> Location = Model.get('stock.location')
    >>> warehouse_loc, = Location.find([('code', '=', 'WH')])
    >>> supplier_loc, = Location.find([('code', '=', 'SUP')])
    >>> customer_loc, = Location.find([('code', '=', 'CUS')])
    >>> output_loc, = Location.find([('code', '=', 'OUT')])
    >>> storage_loc, = Location.find([('code', '=', 'STO')])

Sale Sales::

    >>> Sale = Model.get('sale.sale')
    >>> SaleLine = Model.get('sale.line')
    >>> sale = Sale()
    >>> sale.party = customer
    >>> sale.payment_term = payment_term
    >>> sale.shipment_method = 'order'
    >>> sale_line = SaleLine()
    >>> sale.lines.append(sale_line)
    >>> sale_line.product = product1
    >>> sale_line.quantity = 2.0
    >>> sale_line = SaleLine()
    >>> sale.lines.append(sale_line)
    >>> sale_line.product = product2
    >>> sale_line.quantity = 3.0
    >>> sale.click('quote')
    >>> sale.click('confirm')
    >>> sale.click('process')

    >>> sale = Sale()
    >>> sale.party = customer
    >>> sale.payment_term = payment_term
    >>> sale.shipment_method = 'order'
    >>> sale_line = SaleLine()
    >>> sale.lines.append(sale_line)
    >>> sale_line.product = product1
    >>> sale_line.quantity = 1.0
    >>> sale.click('quote')
    >>> sale.click('confirm')
    >>> sale.click('process')

    >>> sale = Sale()
    >>> sale.party = customer
    >>> sale.payment_term = payment_term
    >>> sale.shipment_method = 'order'
    >>> sale_line = SaleLine()
    >>> sale.lines.append(sale_line)
    >>> sale_line.product = product2
    >>> sale_line.quantity = 1.0
    >>> sale.click('quote')
    >>> sale.click('confirm')
    >>> sale.click('process')

Make 1 unit of the product available::

    >>> StockMove = Model.get('stock.move')
    >>> incoming_move = StockMove()
    >>> incoming_move.product = product1
    >>> incoming_move.uom = unit
    >>> incoming_move.quantity = 100
    >>> incoming_move.from_location = supplier_loc
    >>> incoming_move.to_location = storage_loc
    >>> incoming_move.planned_date = today
    >>> incoming_move.effective_date = today
    >>> incoming_move.company = company
    >>> incoming_move.unit_price = Decimal('1')
    >>> incoming_move.currency = company.currency
    >>> incoming_move.click('do')

    >>> incoming_move = StockMove()
    >>> incoming_move.product = product2
    >>> incoming_move.uom = unit
    >>> incoming_move.quantity = 100
    >>> incoming_move.from_location = supplier_loc
    >>> incoming_move.to_location = storage_loc
    >>> incoming_move.planned_date = today
    >>> incoming_move.effective_date = today
    >>> incoming_move.company = company
    >>> incoming_move.unit_price = Decimal('1')
    >>> incoming_move.currency = company.currency
    >>> incoming_move.click('do')

Assign Shipments Out::

    >>> ShipmentOut = Model.get('stock.shipment.out')
    >>> shipment1, shipment2, shipment3 = ShipmentOut.find([])
    >>> shipment1.click('assign_try')
    True
    >>> shipment2.click('assign_try')
    True
    >>> shipment3.click('assign_try')
    True

Shipment Out Scanning::

    >>> scanning = Wizard('stock.shipment.out.scanning')
    >>> scanning.form.product = product1
    >>> scanning.execute('packed')
    >>> scanning.form.shipment == shipment2
    True

    >>> scanning = Wizard('stock.shipment.out.scanning')
    >>> scanning.form.product = product2
    >>> scanning.execute('packed')
    >>> scanning.form.shipment == shipment1
    True

    >>> shipment1.reload()
    >>> shipment2.reload()
    >>> shipment3.reload()
    >>> shipment1.state
    'done'
    >>> shipment2.state
    'done'
    >>> shipment3.state
    'assigned'
