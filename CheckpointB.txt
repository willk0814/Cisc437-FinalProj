-----------------------------
------  Checkpoint B --------
-----------------------------

--- Question #1 ---
declare
  l_sql varchar2(32767);
  l_col_size number;
  procedure run(p_sql varchar2) as
  begin
    execute immediate p_sql;
  end;
begin
 select data_length into l_col_size from sys.all_tab_cols where owner = 'WILLK' and table_name = 'ORDERS' and column_name = 'ORDER_STATUS';
  
  run('create table ORDERS_NORM (ORDERS_NORM_PK number not null primary key, ORDER_STATUS varchar2('||l_col_size||') not null)');

  run('create sequence ORDER_STATUS_SEQ');

  run('create or replace trigger "T_ORDER_STATUS_TRG"'||
           ' before insert or update on '||
           ' ORDERS_NORM for each row '||
           ' begin '||
           ' if inserting and :new.ORDERS_NORM_PK is null then '||
           '  for c1 in (select ORDER_STATUS_SEQ.nextval nv from dual) loop '||
           '     :new.ORDERS_NORM_PK := c1.nv; '||
           '  end loop; '||
           'end if; '||
           'end;');
 
   run('insert into ORDERS_NORM( ORDER_STATUS ) select distinct ORDER_STATUS from "WILLK"."ORDERS" where ORDER_STATUS is not null');

   run('alter table "ORDERS" add ORDER_STATUS_ID number');

   run('update "ORDERS" x set ORDER_STATUS_ID = (select ORDERS_NORM_PK from ORDERS_NORM where ORDER_STATUS = x.ORDER_STATUS)');

   run('alter table "ORDERS" drop column ORDER_STATUS');

   run('alter table "ORDERS" rename column ORDER_STATUS_ID  to ORDER_STATUS');

   run('alter table "ORDERS" add foreign key (ORDER_STATUS) references ORDERS_NORM(ORDERS_NORM_PK)');
end;
/

----- Question #2 ------

ALTER TABLE customers ADD (
    first_name  VARCHAR2(50),
    last_name   VARCHAR2(50)
);

DECLARE
    CURSOR customers_c IS
    SELECT
        full_name,
        customer_id
    FROM
        customers;

BEGIN
    FOR customers_r IN customers_c LOOP
        UPDATE customers
        SET
            first_name = substr(customers_r.full_name, 0, instr(customers_r.full_name, ' ') - 1),
            last_name = substr(customers_r.full_name, instr(customers_r.full_name, ' ') + 1)
        WHERE
            customers_r.customer_id = customer_id;

    END LOOP;
END;
/
ALTER table customers drop column full_name;
/

----- Question #3A ------
ALTER TABLE products ADD (
    product_color  VARCHAR2(20)
);

DECLARE
    CURSOR colors_c IS
    SELECT
        product_name,
        product_id
    FROM
        products;

BEGIN
    FOR colors_r IN colors_c LOOP
        UPDATE products
        SET
            product_color = RTRIM(substr(colors_r.product_name, instr(colors_r.product_name, '(') + 1), ')'),
            product_name = substr(product_name, 0, instr(product_name, '(') - 1)
        WHERE
            colors_r.product_id = product_id;

    END LOOP;
END;
/

declare
  l_sql varchar2(32767);
  l_col_size number;
  procedure run(p_sql varchar2) as
  begin
    execute immediate p_sql;
  end;
begin
 select data_length into l_col_size from sys.all_tab_cols where owner = 'NMSPILL' and table_name = 'PRODUCTS' and column_name = 'PRODUCT_COLOR';
  
  run('create table ProductColor (ProductColorID number not null primary key, PRODUCT_COLOR varchar2('||l_col_size||') not null)');

  run('create sequence ProductColor_SEQ');

  run('create or replace trigger "T_ProductColor_TRG"'||
           ' before insert or update on '||
           ' ProductColor for each row '||
           ' begin '||
           ' if inserting and :new.ProductColorID is null then '||
           '  for c1 in (select ProductColor_SEQ.nextval nv from dual) loop '||
           '     :new.ProductColorID := c1.nv; '||
           '  end loop; '||
           'end if; '||
           'end;');
 
   run('insert into ProductColor( PRODUCT_COLOR ) select distinct PRODUCT_COLOR from "NMSPILL"."PRODUCTS" where PRODUCT_COLOR is not null');

   run('alter table "PRODUCTS" add PRODUCT_COLOR_ID number');

   run('update "PRODUCTS" x set PRODUCT_COLOR_ID = (select ProductColorID from ProductColor where PRODUCT_COLOR = x.PRODUCT_COLOR)');

   run('alter table "PRODUCTS" drop column PRODUCT_COLOR');

   run('alter table "PRODUCTS" rename column PRODUCT_COLOR_ID  to PRODUCT_COLOR');

   run('alter table "PRODUCTS" add foreign key (PRODUCT_COLOR) references ProductColor(ProductColorID)');
end;
/


----- Question #3B ------
CREATE TABLE PRODUCTPRICE 
(
  PRODUCT_ID NUMBER(*,0) NOT NULL 
, EFFECTIVE_DATE DATE NOT NULL 
, UNIT_PRICE NUMBER(10,2) NOT NULL 
);
/

INSERT INTO productprice (
        product_id,
        effective_date,
        unit_price
    )
        SELECT
            product_id,
            sysdate,
            unit_price
        FROM
            products
/

----- Question #4 ------
create or replace PROCEDURE prc_increase_price (
    product_id_input        IN  NUMBER,
    effective_date_input    IN  DATE,
    percent_increase_input  IN  NUMBER
) AS
BEGIN
    DECLARE
        CURSOR productprice_c IS
        SELECT DISTINCT
            product_id
        FROM
            productprice;

    BEGIN
        IF
            product_id_input IS NULL
            AND effective_date_input IS NULL
        THEN
            FOR productprice_i IN productprice_c LOOP
                INSERT INTO productprice VALUES (
                    productprice_i.product_id,
                    (
                        SELECT
                            MAX(effective_date) + 1
                        FROM
                            productprice
                        WHERE
                            productprice_i.product_id = product_id_input
                        GROUP BY
                            product_id
                    ),
                    (
                        SELECT
                            unit_price
                        FROM
                            productprice
                        WHERE
                                productprice_i.product_id = product_id_input
                            AND effective_date = (
                                SELECT
                                    MAX(effective_date)
                                FROM
                                    productprice
                                WHERE
                                    productprice_i.product_id = product_id_input
                                GROUP BY
                                    product_id
                            )
                    ) * ( 1 + percent_increase_input )
                );

            END LOOP;

        ELSIF
            product_id_input IS NULL
            AND effective_date_input IS NOT NULL
        THEN
            UPDATE productprice
            SET
                unit_price = unit_price + ( unit_price * percent_increase_input );

        ELSIF
            effective_date_input IS NULL
            AND product_id_input IS NOT NULL
        THEN
            INSERT INTO productprice VALUES (
                product_id_input,
                (
                    SELECT
                        MAX(effective_date) + 1
                    FROM
                        productprice
                    WHERE
                        product_id = product_id_input
                    GROUP BY
                        product_id
                ),
                (
                    SELECT
                        unit_price
                    FROM
                        productprice
                    WHERE
                            product_id = product_id_input
                        AND effective_date = (
                            SELECT
                                MAX(effective_date)
                            FROM
                                productprice
                            WHERE
                                product_id = product_id_input
                            GROUP BY
                                product_id
                        )
                ) * ( 1 + percent_increase_input )
            );

        ELSE
            INSERT INTO productprice VALUES (
                product_id_input,
                effective_date_input,
                (
                        SELECT
                            unit_price
                        FROM
                            productprice
                        WHERE
                                product_id = product_id_input
                            AND effective_date = (
                                SELECT
                                    MAX(effective_date)
                                FROM
                                    productprice
                                WHERE
                                    product_id = product_id_input
                                GROUP BY
                                    product_id
                            )
                    ) * ( 1 + percent_increase_input )
                );
        END IF;

    END;
END prc_increase_price;


----- Question #5 ------
CREATE VIEW v_product_price AS
    SELECT
        product_id,
        effective_date AS start_date,
        LEAD(EFFECTIVE_DATE) OVER (
        PARTITION BY PRODUCT_ID
        ORDER BY EFFECTIVE_DATE    
    ) END_DATE
    FROM
        PRODUCT_PRICE;
/

----- Question #6 ------
DECLARE
    v_max  NUMBER(38,0);
    v_sql  VARCHAR2(2000);
BEGIN
    SELECT
        MAX(customer_id) + 1
    INTO v_max
    FROM
        customers;

    v_sql := 'CREATE SEQUENCE CUSTOMER_SEQ INCREMENT BY 1 START WITH '
             || v_max
             || ' MAXVALUE  99999999 MINVALUE 1';
    
    EXECUTE IMMEDIATE v_sql;
    
END;
/

DECLARE
    v_max  NUMBER(38,0);
    v_sql  VARCHAR2(2000);
BEGIN
    SELECT
        MAX(order_id) + 1
    INTO v_max
    FROM
        orders;

    v_sql := 'CREATE SEQUENCE order_SEQ INCREMENT BY 1 START WITH '
             || v_max
             || ' MAXVALUE  99999999 MINVALUE 1';
    
    EXECUTE IMMEDIATE v_sql;
    
END;
/

DECLARE
    v_max  NUMBER(38,0);
    v_sql  VARCHAR2(2000);
BEGIN
    SELECT
        MAX(product_id) + 1
    INTO v_max
    FROM
        products;

    v_sql := 'CREATE SEQUENCE products_SEQ INCREMENT BY 1 START WITH '
             || v_max
             || ' MAXVALUE  99999999 MINVALUE 1';
    
    EXECUTE IMMEDIATE v_sql;
    
END;
/

DECLARE
    v_max  NUMBER(38,0);
    v_sql  VARCHAR2(2000);
BEGIN
    SELECT
        MAX(store_id) + 1
    INTO v_max
    FROM
        stores;

    v_sql := 'CREATE SEQUENCE stores_SEQ INCREMENT BY 1 START WITH '
             || v_max
             || ' MAXVALUE  99999999 MINVALUE 1';
    
    EXECUTE IMMEDIATE v_sql;
    
END;
/

CREATE OR REPLACE TRIGGER customer_seq_trg BEFORE
    INSERT ON customers
    FOR EACH ROW
BEGIN
    SELECT
        customer_seq.NEXTVAL
    INTO :new.customer_id
    FROM
        dual;
END;
/

CREATE OR REPLACE TRIGGER order_seq_trg BEFORE
    INSERT ON orders
    FOR EACH ROW
BEGIN
    SELECT
        order_seq.NEXTVAL
    INTO :new.order_id
    FROM
        dual;
END;
/

CREATE OR REPLACE TRIGGER stores_seq_trg BEFORE
    INSERT ON stores
    FOR EACH ROW
BEGIN
    SELECT
        stores_seq.NEXTVAL
    INTO :new.store_id
    FROM
        dual;
END;
/

CREATE OR REPLACE TRIGGER products_seq_trg BEFORE
    INSERT ON products
    FOR EACH ROW
BEGIN
    SELECT
        products_seq.NEXTVAL
    INTO :new.product_id
    FROM
        dual;
END;
/


----- Question #7 ------
ALTER TABLE products ADD (
    Createdby  VARCHAR2(215),
    modifiedby VARCHAR2(215)
);
/
ALTER TABLE customers ADD (
    Createdby  VARCHAR2(215),
    modifiedby VARCHAR2(215)
);
/
ALTER TABLE orders ADD (
    Createdby  VARCHAR2(215),
    modifiedby VARCHAR2(215)
);
/
ALTER TABLE stores ADD (
    Createdby  VARCHAR2(215),
    modifiedby VARCHAR2(215)
);
/
ALTER TABLE orders_norm ADD (
    Createdby  VARCHAR2(215),
    modifiedby VARCHAR2(215)
);
/
ALTER TABLE productcolor ADD (
    Createdby  VARCHAR2(215),
    modifiedby VARCHAR2(215)
);
/
ALTER TABLE productprice ADD (
    Createdby  VARCHAR2(215),
    modifiedby VARCHAR2(215)
);
/
ALTER TABLE order_items ADD (
    Createdby  VARCHAR2(215),
    modifiedby VARCHAR2(215)
);
/

CREATE OR REPLACE TRIGGER CUSTOMERS_FP_TRG BEFORE
    INSERT OR UPDATE ON customers
    FOR EACH ROW
DECLARE
    nr_user VARCHAR(40);
BEGIN
    SELECT
        user
    INTO nr_user
    FROM
        dual;

    IF inserting THEN
        :new.createdby := nr_user;
        :new.modifiedby := nr_user;
    END IF;

    IF updating THEN
        :new.modifiedby := nr_user;
    END IF;
END;
/

CREATE OR REPLACE TRIGGER PRODUCTS_FP_TRG BEFORE
    INSERT OR UPDATE ON products
    FOR EACH ROW
DECLARE
    nr_user VARCHAR(40);
BEGIN
    SELECT
        user
    INTO nr_user
    FROM
        dual;

    IF inserting THEN
        :new.createdby := nr_user;
        :new.modifiedby := nr_user;
    END IF;

    IF updating THEN
        :new.modifiedby := nr_user;
    END IF;
END;
/

CREATE OR REPLACE TRIGGER ORDERS_FP_TRG BEFORE
    INSERT OR UPDATE ON orders
    FOR EACH ROW
DECLARE
    nr_user VARCHAR(40);
BEGIN
    SELECT
        user
    INTO nr_user
    FROM
        dual;

    IF inserting THEN
        :new.createdby := nr_user;
        :new.modifiedby := nr_user;
    END IF;

    IF updating THEN
        :new.modifiedby := nr_user;
    END IF;
END;
/

CREATE OR REPLACE TRIGGER ORDERS_NORM_FP_TRG BEFORE
    INSERT OR UPDATE ON ORDERS_NORM
    FOR EACH ROW
DECLARE
    nr_user VARCHAR(40);
BEGIN
    SELECT
        user
    INTO nr_user
    FROM
        dual;

    IF inserting THEN
        :new.createdby := nr_user;
        :new.modifiedby := nr_user;
    END IF;

    IF updating THEN
        :new.modifiedBy := nr_user;
    END IF;
END;
/
CREATE OR REPLACE TRIGGER STORES_FP_TRG BEFORE
    INSERT OR UPDATE ON stores
    FOR EACH ROW
DECLARE
    nr_user VARCHAR(40);
BEGIN
    SELECT
        user
    INTO nr_user
    FROM
        dual;

    IF inserting THEN
        :new.createdby := nr_user;
        :new.modifiedby := nr_user;
    END IF;

    IF updating THEN
        :new.modifiedBy := nr_user;
    END IF;
END;
/

CREATE OR REPLACE TRIGGER PRODUCTCOLOR_FP_TRG BEFORE
    INSERT OR UPDATE ON PRODUCTCOLOR
    FOR EACH ROW
DECLARE
    nr_user VARCHAR(40);
BEGIN
    SELECT
        user
    INTO nr_user
    FROM
        dual;

    IF inserting THEN
        :new.createdby := nr_user;
        :new.modifiedby := nr_user;
    END IF;

    IF updating THEN
        :new.modifiedBy := nr_user;
    END IF;
END;
/

CREATE OR REPLACE TRIGGER PRODUCTPRICE_FP_TRG BEFORE
    INSERT OR UPDATE ON stores
    FOR EACH ROW
DECLARE
    nr_user VARCHAR(40);
BEGIN
    SELECT
        user
    INTO nr_user
    FROM
        dual;

    IF inserting THEN
        :new.createdby := nr_user;
        :new.modifiedby := nr_user;
    END IF;

    IF updating THEN
        :new.modifiedBy := nr_user;
    END IF;
END;
/

CREATE OR REPLACE TRIGGER ORDER_ITEMS_FP_TRG BEFORE
    INSERT OR UPDATE ON ORDER_ITEMS
    FOR EACH ROW
DECLARE
    nr_user VARCHAR(40);
BEGIN
    SELECT
        user
    INTO nr_user
    FROM
        dual;

    IF inserting THEN
        :new.createdby := nr_user;
        :new.modifiedby := nr_user;
    END IF;

    IF updating THEN
        :new.modifiedBy := nr_user;
    END IF;
END;
/


