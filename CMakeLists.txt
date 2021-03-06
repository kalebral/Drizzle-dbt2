CMAKE_MINIMUM_REQUIRED(VERSION 2.4)
PROJECT(DBT2 C)

SET(abs_top_srcdir ${CMAKE_HOME_DIRECTORY})
MESSAGE(STATUS "Current working directory is : ${abs_top_srcdir}")

#
# Defining binaries so to minimized typoes.
#

SET(DBT2_CLIENT bin/dbt2-client)
SET(DBT2_DATAGEN bin/dbt2-datagen)
SET(DBT2_DRIVER bin/dbt2-driver)
SET(DBT2_TXN_TEST bin/dbt2-transaction-test)

#
# Check for large file support by using 'getconf'.
#

FIND_PROGRAM(GETCONF getconf)
IF (GETCONF)
    SET(GETCONF_FOUND TRUE)
ENDIF (GETCONF)

IF (GETCONF_FOUND)
    EXECUTE_PROCESS(COMMAND ${GETCONF} LFS_CFLAGS COMMAND ${GETCONF}
        LFS64_CFLAGS OUTPUT_VARIABLE LFS_CFLAGS
        OUTPUT_STRIP_TRAILING_WHITESPACE)
ENDIF (GETCONF_FOUND)

#
# Create PostgreSQL specific configuration.
#

IF (WITH-DB STREQUAL "postgres")
    FIND_PROGRAM(PG_CONFIG pg_config)

    IF (PG_CONFIG)
       SET(PG_CONFIG_FOUND TRUE)
    ENDIF (PG_CONFIG)

    IF (PG_CONFIG_FOUND)
        MESSAGE(STATUS "Found pg_config for PostgreSQL: ${PG_CONFIG}")
        SET(LIBPQ_SOURCE src/libpq/dbc_common.c src/libpq/dbc_new_order.c
            src/libpq/dbc_payment.c src/libpq/dbc_order_status.c
            src/libpq/dbc_delivery.c src/libpq/dbc_stock_level.c
            src/libpq/dbc_integrity.c)
        EXECUTE_PROCESS(COMMAND ${PG_CONFIG} --includedir OUTPUT_VARIABLE
            PG_INCLUDEDIR OUTPUT_STRIP_TRAILING_WHITESPACE)
        EXECUTE_PROCESS(COMMAND ${PG_CONFIG} --libdir OUTPUT_VARIABLE
            PG_LIBDIR OUTPUT_STRIP_TRAILING_WHITESPACE)
        SET(LIBPQ_FLAGS "-DLIBPQ -I${PG_INCLUDEDIR}")
        SET(LIBPQ_LIBS "-L${PG_LIBDIR} -lpq")
        SET(PGSQL_SCRIPTS bin/pgsql/dbt2-pgsql-analyze-stats
            bin/pgsql/dbt2-pgsql-build-db bin/pgsql/dbt2-pgsql-check-db
            bin/pgsql/dbt2-pgsql-create-db bin/pgsql/dbt2-pgsql-create-indexes
            bin/pgsql/dbt2-pgsql-create-tables bin/pgsql/dbt2-pgsql-db-stat
            bin/pgsql/dbt2-pgsql-drop-db bin/pgsql/dbt2-pgsql-drop-tables
            bin/pgsql/dbt2-pgsql-load-db bin/pgsql/dbt2-pgsql-load-stored-procs
            bin/pgsql/dbt2-pgsql-plans bin/pgsql/dbt2-pgsql-start-db
            bin/pgsql/dbt2-pgsql-stop-db)
    ENDIF (PG_CONFIG_FOUND)

#
# Create MySQL specific configuration.
#
ELSEIF (WITH-DB STREQUAL "mysql")
    FIND_PROGRAM(MYSQL_CONFIG mysql_config)

    IF (MYSQL_CONFIG)
        SET(MYSQL_CONFIG_FOUND TRUE)
    ENDIF (MYSQL_CONFIG)

    IF(NOT DEFINED MYSQL)
        SET(MYSQL /usr/bin/sql )
    ENDIF(NOT DEFINED MYSQL)

    IF (MYSQL_CONFIG_FOUND)
        MESSAGE(STATUS "Found mysql_config for MySQL: ${MYSQL_CONFIG}")
        SET(MYSQL_SOURCE src/mysql/dbc_common.c src/mysql/dbc_new_order.c
            src/mysql/dbc_payment.c src/mysql/dbc_order_status.c
            src/mysql/dbc_delivery.c src/mysql/dbc_stock_level.c
            src/mysql/dbc_integrity.c)
        EXECUTE_PROCESS(COMMAND ${MYSQL_CONFIG} --include OUTPUT_VARIABLE
            MYSQL_INCLUDEDIR OUTPUT_STRIP_TRAILING_WHITESPACE)
        EXECUTE_PROCESS(COMMAND ${MYSQL_CONFIG} --libs OUTPUT_VARIABLE
        MYSQL_LIBS OUTPUT_STRIP_TRAILING_WHITESPACE)
        SET(MYSQL_FLAGS "-DLIBMYSQL ${MYSQL_INCLUDEDIR}")
        CONFIGURE_FILE(bin/mysql/dbt2-mysql-build-db.in  bin/mysql/dbt2-mysql-build-db  @ONLY)
        CONFIGURE_FILE(bin/mysql/dbt2-mysql-plans.in  bin/mysql/dbt2-mysql-plans  @ONLY)
        CONFIGURE_FILE(bin/mysql/dbt2-mysql-start-db.in  bin/mysql/dbt2-mysql-start-db  @ONLY)
        CONFIGURE_FILE(bin/mysql/dbt2-mysql-stop-db.in  bin/mysql/dbt2-mysql-stop-db  @ONLY)
        SET(MYSQL_SCRIPTS
            bin/mysql/dbt2-mysql-build-db
            bin/mysql/dbt2-mysql-plans
            bin/mysql/dbt2-mysql-start-db
            bin/mysql/dbt2-mysql-stop-db)
    ENDIF (MYSQL_CONFIG_FOUND)

#
# Create Drizzle configuration
#
ELSEIF (WITH-DB STREQUAL "drizzle")

    SET(DRIZZLE_SOURCE src/drizzle/dbc_common.c
                       src/nonsp/dbc_new_order.c
                       src/nonsp/dbc_payment.c
                       src/nonsp/dbc_order_status.c
                       src/nonsp/dbc_delivery.c
                       src/nonsp/dbc_stock_level.c
                       src/nonsp/dbc_nonsp.c
                       src/nonsp/dbc_integrity.c)

    IF(NOT DEFINED DRIZZLE_INCLUDEDIR)
        SET(DRIZZLE_INCLUDEDIR /usr/local/include )
    ENDIF(NOT DEFINED DRIZZLE_INCLUDEDIR)

    IF(NOT DEFINED DRIZZLE_LIBS)
        SET(DRIZZLE_LIBS "-ldrizzle  -lpthread")
    ENDIF(NOT DEFINED DRIZZLE_LIBS)

    SET(DRIZZLE_FLAGS "-DLIBDRIZZLE -I${DRIZZLE_INCLUDEDIR}")

    MESSAGE(STATUS "Drizzle configuration build flags :  ${DRIZZLE_FLAGS}")
    MESSAGE(STATUS "Drizzle configuration library flags:  ${DRIZZLE_LIBS}")

    CONFIGURE_FILE(bin/drizzle/dbt2-drizzle-start-db.in  bin/dbt2-drizzle-start-db  @ONLY)
    CONFIGURE_FILE(bin/drizzle/dbt2-drizzle-stop-db.in  bin/dbt2-drizzle-stop-db  @ONLY)
    SET(DRIZZLE_SCRIPTS bin/drizzle/dbt2-drizzle_admin dbt2-drizzle-load-db)


ELSE (NOT DEFINED WITH-DB)
    MESSAGE(FATAL_ERROR "Need to specify -DWITH-DB=<drizzle>|<mysql>|<postgres>")

ENDIF (WITH-DB STREQUAL "postgres")

#
# Defining commong groups of source files.
#

SET(COMMON_SOURCE src/common.c src/logging.c src/_socket.c
    src/client_interface.c src/transaction_data.c src/_semaphore.c)

#
# Files to build.
#

ADD_EXECUTABLE(${DBT2_CLIENT} src/client.c src/transaction_queue.c
    src/listener.c src/db_threadpool.c src/db.c ${COMMON_SOURCE}
    ${LIBPQ_SOURCE} ${MYSQL_SOURCE} ${DRIZZLE_SOURCE})
ADD_EXECUTABLE(${DBT2_DATAGEN} src/datagen.c src/common.c)
ADD_EXECUTABLE(${DBT2_DRIVER} ${COMMON_SOURCE} src/driver_main.c src/driver.c
    src/input_data_generator.c)
ADD_EXECUTABLE(${DBT2_TXN_TEST} src/transaction_test.c src/db.c
    src/input_data_generator.c ${COMMON_SOURCE} ${LIBPQ_SOURCE} ${MYSQL_SOURCE} ${DRIZZLE_SOURCE})

#
# Set compiler flags.
#

SET(CMAKE_C_FLAGS "-Wall -Isrc/include ${LFS_CFLAGS}")

SET_SOURCE_FILES_PROPERTIES(src/db_threadpool.c src/db.c src/client.c
    src/transaction_test.c  src/common.c
    COMPILE_FLAGS "${LIBPQ_FLAGS} ${MYSQL_FLAGS} ${DRIZZLE_FLAGS}")
SET_SOURCE_FILES_PROPERTIES(${LIBPQ_SOURCE}
    COMPILE_FLAGS "${LIBPQ_FLAGS}")
SET_SOURCE_FILES_PROPERTIES(${MYSQL_SOURCE}
    COMPILE_FLAGS "${MYSQL_FLAGS}")
SET_SOURCE_FILES_PROPERTIES(${DRIZZLE_SOURCE}
    COMPILE_FLAGS "${DRIZZLE_FLAGS}")

#
# Set linker flags.
#

SET_TARGET_PROPERTIES(${DBT2_CLIENT} PROPERTIES LINK_FLAGS "-lm ${LIBPQ_LIBS} ${MYSQL_LIBS} ${DRIZZLE_LIBS}")
SET_TARGET_PROPERTIES(${DBT2_DATAGEN} PROPERTIES LINK_FLAGS -lm)
SET_TARGET_PROPERTIES(${DBT2_DRIVER} PROPERTIES LINK_FLAGS "-lm -lpthread")
SET_TARGET_PROPERTIES(${DBT2_TXN_TEST} PROPERTIES LINK_FLAGS "-lm ${LIBPQ_LIBS} ${MYSQL_LIBS} ${DRIZZLE_LIBS}")

#
# Install rules.
#

INSTALL(PROGRAMS ${DBT2_CLIENT} ${DBT2_DATAGEN} ${DBT2_DRIVER} ${DBT2_TXN_TEST}
    bin/dbt2-generate-report bin/dbt2-get-os-info bin/dbt2-post-process
    bin/dbt2-run-workload bin/dbt2-sysstats
    ${PGSQL_SCRIPTS} ${MYSQL_SCRIPTS} DESTINATION "/bin")
