<div class="modal-header">
  <h4 class="modal-title">Add Contact</h4>
  <button type="button" class="btn-close" aria-label="Close" (click)="close()"></button>
</div>
<div class="modal-body">
  <form>
    <div class="mb-3">
      <label for="name" class="form-label">Name</label>
      <input type="text" id="name" class="form-control" [(ngModel)]="contact.name" name="name" required />
    </div>
    <div class="mb-3">
      <label for="phone" class="form-label">Phone</label>
      <input type="text" id="phone" class="form-control" [(ngModel)]="contact.phone" name="phone" required />
    </div>
  </form>
</div>
<div class="modal-footer">
  <button type="button" class="btn btn-primary" (click)="saveContact()">Save</button>
  <button type="button" class="btn btn-secondary" (click)="close()">Close</button>
</div>
