# Zaliczenie_KalkowskaKlaudia
#-*- codingutf-8 -*-

import sqlite3
from datetime import datetime

db_path = 'zaliczenie1.db'

#
# Wyjatek uzywany w repozytorium
#
class RepositoryException(Exception):
    def __init__(self, message, *errors):
        Exception.__init__(self, message)
        self.errors = errors


#
# Model danych
#
class Karta():
    """Model pojedynczej karty
    """
    def __init__(self, id, date=datetime.now(), pozycje=[]):
        self.id = id
        self.date = date
        self.pozycje = pozycje
        self.amount = sum([item.qty for item in self.pozycje])

    def __repr__(self):
        return "<Karta(id='%s', date='%s', amount='%s', pozycje='%s')>" % (
                    self.id, self.date, str(self.amount), str(self.pozycje)
                )


class Pozycje():
    """Model pozycji na karcie. Wystepuje tylko wewnatrz obiektu Karta.
    """
    def __init__(self, author, title,kind,  qty):
        self.author = author
        self.title = title
        self.kind = kind
        #self.amount = amount
        self.qty = qty

    def __repr__(self):
        return "<Pozycje(author='%s', title='%s',kind='%s', qty='%s')>" % (
                    self.author, self.title, self.kind, str(self.qty)
                )


#
# Klasa bazowa repozytorium
#
class Repository():
    def __init__(self):
        try:
            self.conn = self.get_connection()
        except Exception as e:
            raise RepositoryException('GET CONNECTION:', *e.args)
        self._complete = False

    # wejscie do with ... as ...
    def __enter__(self):
        return self

    # wyjscie z with ... as ...
    def __exit__(self, type_, value, traceback):
        self.close()

    def complete(self):
        self._complete = True

    def get_connection(self):
        return sqlite3.connect(db_path)

    def close(self):
        if self.conn:
            try:
                if self._complete:
                    self.conn.commit()
                else:
                    self.conn.rollback()
            except Exception as e:
                raise RepositoryException(*e.args)
            finally:
                try:
                    self.conn.close()
                except Exception as e:
                    raise RepositoryException(*e.args)

#
# repozytorium obiektow typu Invoice
#
class KartaRepository(Repository):

    def add(self, karta):
        """Metoda dodaje pojedyncza karte do bazy danych,
        wraz ze wszystkimi jej pozycjami.
        """
        try:
            c = self.conn.cursor()
            # zapisz naglowek faktury
            amount = sum([item.qty for item in karta.pozycje])
            c.execute('INSERT INTO Karta (id, inv_date, amount) VALUES(?, ?,?)',
                        (karta.id, str(karta.date), karta.amount)
                    )
            # zapisz pozycje faktury
            if karta.pozycje:
                for invitem in karta.pozycje:
                    try:
                        c.execute('INSERT INTO Pozycje (author, title, kind, qty, karta_id) VALUES(?,?,?,?,?)',
                                        (invitem.author, invitem.title,invitem.kind, invitem.qty, karta.id)
                                )
                    except Exception as e:
                        #print "item add error:", e
                        raise RepositoryException('error adding karta item: %s, to karta: %s'
                                                    %( str(invitem), str(karta.id))
                                                )
        except Exception as e:
            #print "karta add error:", e
            raise RepositoryException('error adding karta %s' % str(karta))

    def delete(self, karta):
        """Metoda usuwa pojedyncza karte z bazy danych,
        wraz ze wszystkimi jej pozycjami.
        """
        try:
            c = self.conn.cursor()
            # usun pozycje
            c.execute('DELETE FROM Pozycje WHERE karta_id=?', (karta.id,))
            # usun naglowek
            c.execute('DELETE FROM Karta WHERE id=?', (karta.id,))

        except Exception as e:
            #print "invoice delete error:", e
            raise RepositoryException('error deleting karta %s' % str(karta))

    def getById(self, id):
        """Get karta by id
        """
        try:
            c = self.conn.cursor()
            c.execute("SELECT * FROM Karta WHERE id=?", (id,))
            inv_row = c.fetchone()
            karta = Karta(id=id)
            if inv_row == None:
                karta=None
            else:
                karta.date = inv_row[1]
                karta.amount = inv_row[2]
                c.execute("SELECT * FROM Pozycje WHERE karta_id=? order by author", (id,))
                inv_items_rows = c.fetchall()
                items_list = []
                for item_row in inv_items_rows:
                    item = Pozycje(author=item_row[0], title=item_row[1], kind=item_row[2], qty=item_row[3])
                    items_list.append(item)
                karta.pozycje=items_list
        except Exception as e:
            #print "invoice getById error:", e
            raise RepositoryException('error getting by id karta_id: %s' % str(id))
        return karta

    def update(self, karta):
        """Metoda uaktualnia pojedyncza karte w bazie danych,
        wraz ze wszystkimi jej pozycjami.
        """
        try:
            # pobierz z bazy fakture
            inv_oryg = self.getById(karta.id)
            if inv_oryg != None:
                # faktura jest w bazie: usun ja
                self.delete(karta)
            self.add(karta)

        except Exception as e:
            #print "invoice update error:", e
            raise RepositoryException('error updating karta %s' % str(karta))
if __name__ == '__main__':
    try:
        with KartaRepository() as karta_repository:
            karta_repository.add(
                Karta(id = 3, date = datetime.now(),
                      pozycje = [
                            Pozycje(author = "Nicholas Sparks", title="I wciaz ja kocham",kind="melodramat" , qty =1),
                            Pozycje(author = "Jane Austen",  title="Mansfield Park" ,kind="" , qty = 1),
                            Pozycje(author = "Marta Guzowska", title="Wszyscy ludzie przezcaly czas", kind="kryminal" , qty = 1),

                      ]
                   )
                )
            karta_repository.complete()
            karta_repository.add(
                Karta(id = 2, date = datetime.now(),
                      pozycje = [
                            Pozycje(author = "Agata Christie", title="Dom zbrodni" ,kind="krymianal" , qty = 1),
                            Pozycje(author = "Lisa Unger",  title="Wyspa nieprawdy" ,kind="thriller" , qty = 2),
                            Pozycje(author = "Stephen King", title="Martwa Strefa" ,kind="thriller" , qty = 3),
                            Pozycje(author= "Marta Guzowska" , title="Glowa Niobe", kind="kryminal", qty=1),
                      ]
                   )
                )
            karta_repository.complete()
            karta_repository.add(
                Karta(id = 5, date = datetime.now(),
                      pozycje = [
                            Pozycje(author = "Stephen King", title="Podpalaczka" ,kind="thriller" ,  qty = 11),
                            Pozycje(author = "Carole Matthews",  title="Klub milosniczek czekolady" , kind="komedia" ,  qty = 3),
                            Pozycje(author = "Nicholas Sparks", title="Bezpieczna przystan" ,kind="melodramat" , qty = 5),
                      ]
                   )
                )
            karta_repository.complete()

    except RepositoryException as e:
        print(e)
    print KartaRepository().getById(1)

    try:
        with KartaRepository() as karta_repository:
            karta_repository.update(
                Karta(id = 3, date = datetime.now(),
                        pozycje = [
                            Pozycje(author = "Roz Bailey", title="Imprezowe dziewczyny" ,kind="komedia" ,  qty = 10),
                            Pozycje(author = "Agata Christie",  title="Zerwane zareczyny" , kind="kryminal" , qty = 3),
                            Pozycje(author = "Dennis Lehane", title="Gdzie jestes Amando" ,kind="kryminal" , qty = 5),
                            Pozycje(author = "Tess Gerritsen", title="Ogrod kosci" ,kind="thriller" , qty = 5),


                        ]
                    )
                )
            karta_repository.complete()
    except RepositoryException as e:
         print(e)

    print KartaRepository().getById(1)

    try:
         with KartaRepository() as karta_repository:
             karta_repository.delete( Karta(id = 1) )
             karta_repository.complete()
    except RepositoryException as e:
          print(e)
    print KartaRepository().getById(1)
